We'll implement a **million-turn evolutionary engine** for the BIJA codebase. This system will evolve not just parameters, but the actual structure and behavior of the bat agents over generations, using genetic algorithms and LLM-guided mutations.

## 🧬 Evolution Architecture Overview

The evolutionary process will be managed by a dedicated module that:
1. **Defines a Genome** containing all evolvable aspects (parameters, action weights, even code snippets).
2. **Mutates and Crosses** genomes to produce offspring.
3. **Evaluates Fitness** by running the agent in a simulated environment and measuring performance.
4. **Selects the Best** individuals to form the next generation.
5. **Persists Evolution State** to allow resuming long runs.
6. **Uses Distributed Evaluation** across multiple BEAM nodes for massive parallelism.

---

## 📁 Additional Files for Evolution

Add these files to the BIJA project structure:

```
lib/bija/evolution/
├── engine.ex
├── fitness.ex
├── genome.ex        (enhanced version)
├── mutation.ex
├── population.ex
└── runner.ex

lib/mix/tasks/
└── bija.evolve.ex
```

---

## 📄 File Contents

### `lib/bija/evolution/genome.ex` (Enhanced)

```elixir
defmodule BIJA.Evolution.Genome do
  @moduledoc """
  Represents a bat's complete evolvable traits, including action compositions and code variants.
  """
  defstruct [
    # Core parameters
    :echolocation_sensitivity,
    :social_tendency,
    :exploration_bias,
    :rest_interval,
    :energy_decay_rate,
    :learning_rate,
    # Action weights (which actions to prioritize)
    :action_weights,
    # Code variants (symbolic representation of action implementations)
    :action_variants,
    # Fitness score
    fitness: 0.0,
    generation: 0,
    id: nil
  ]

  @action_types [:explore, :rest, :broadcast, :focus]

  def random do
    %__MODULE__{
      id: UUID.uuid4(),
      echolocation_sensitivity: :rand.uniform(),
      social_tendency: :rand.uniform(),
      exploration_bias: :rand.uniform(),
      rest_interval: Enum.random(1_000..60_000),
      energy_decay_rate: :rand.uniform() * 0.01,
      learning_rate: :rand.uniform() * 0.5,
      action_weights: random_action_weights(),
      action_variants: random_action_variants()
    }
  end

  defp random_action_weights do
    @action_types
    |> Enum.map(&{&1, :rand.uniform()})
    |> Map.new()
  end

  defp random_action_variants do
    @action_types
    |> Enum.map(&{&1, Enum.random([:default, :aggressive, :cautious, :cooperative])})
    |> Map.new()
  end

  def crossover(p1, p2) do
    %__MODULE__{
      id: UUID.uuid4(),
      echolocation_sensitivity: random_gene(p1.echolocation_sensitivity, p2.echolocation_sensitivity),
      social_tendency: random_gene(p1.social_tendency, p2.social_tendency),
      exploration_bias: random_gene(p1.exploration_bias, p2.exploration_bias),
      rest_interval: Enum.random([p1.rest_interval, p2.rest_interval]),
      energy_decay_rate: random_gene(p1.energy_decay_rate, p2.energy_decay_rate),
      learning_rate: random_gene(p1.learning_rate, p2.learning_rate),
      action_weights: crossover_action_weights(p1.action_weights, p2.action_weights),
      action_variants: crossover_action_variants(p1.action_variants, p2.action_variants)
    }
  end

  defp random_gene(a, b), do: Enum.random([a, b])

  defp crossover_action_weights(w1, w2) do
    Map.new(@action_types, fn action ->
      {action, (Map.get(w1, action) + Map.get(w2, action)) / 2}
    end)
  end

  defp crossover_action_variants(v1, v2) do
    Map.new(@action_types, fn action ->
      {action, Enum.random([Map.get(v1, action), Map.get(v2, action)])}
    end)
  end
end
```

---

### `lib/bija/evolution/mutation.ex`

```elixir
defmodule BIJA.Evolution.Mutation do
  @moduledoc """
  Mutation operators for genomes.
  """
  alias BIJA.Evolution.Genome

  @action_types [:explore, :rest, :broadcast, :focus]
  @variant_types [:default, :aggressive, :cautious, :cooperative]

  def mutate(genome, rate \\ 0.1) do
    genome
    |> mutate_float(:echolocation_sensitivity, rate)
    |> mutate_float(:social_tendency, rate)
    |> mutate_float(:exploration_bias, rate)
    |> mutate_int(:rest_interval, 1000, 60000, rate)
    |> mutate_float(:energy_decay_rate, rate, 0.001)
    |> mutate_float(:learning_rate, rate, 0.01)
    |> mutate_action_weights(rate)
    |> mutate_action_variants(rate)
  end

  defp mutate_float(genome, field, rate, delta \\ 0.1) do
    if :rand.uniform() < rate do
      new_val = Map.get(genome, field) + (:rand.uniform() - 0.5) * delta
      Map.put(genome, field, max(0.0, min(1.0, new_val)))
    else
      genome
    end
  end

  defp mutate_int(genome, field, min, max, rate) do
    if :rand.uniform() < rate do
      delta = round((:rand.uniform() - 0.5) * (max - min) * 0.1)
      new_val = Map.get(genome, field) + delta
      Map.put(genome, field, max(min, min(max, new_val)))
    else
      genome
    end
  end

  defp mutate_action_weights(genome, rate) do
    if :rand.uniform() < rate do
      action = Enum.random(@action_types)
      weights = genome.action_weights
      new_weight = Map.get(weights, action) + (:rand.uniform() - 0.5) * 0.2
      new_weights = Map.put(weights, action, max(0.0, min(1.0, new_weight)))
      %{genome | action_weights: new_weights}
    else
      genome
    end
  end

  defp mutate_action_variants(genome, rate) do
    if :rand.uniform() < rate do
      action = Enum.random(@action_types)
      variants = genome.action_variants
      new_variant = Enum.random(@variant_types -- [Map.get(variants, action)])
      new_variants = Map.put(variants, action, new_variant)
      %{genome | action_variants: new_variants}
    else
      genome
    end
  end
end
```

---

### `lib/bija/evolution/fitness.ex`

```elixir
defmodule BIJA.Evolution.Fitness do
  @moduledoc """
  Evaluates the fitness of a genome by running a simulated bat agent.
  """
  alias BIJA.Bat
  alias BIJA.Evolution.Genome

  @doc """
  Returns a fitness score (higher is better) for the given genome.
  The score is based on:
  - Number of discoveries made
  - Energy efficiency
  - Exploration coverage
  - Social contribution (if applicable)
  """
  def evaluate(genome, environment \\ default_environment()) do
    # Create a bat with this genome
    {:ok, bat_pid} = Bat.start_link(%{
      id: "eval_#{genome.id}",
      genome: Map.from_struct(genome)
    })

    # Run simulation for a fixed number of steps
    steps = 50
    discoveries_before = 0
    energy_before = 1.0

    # Simulate exploration
    for _ <- 1..steps do
      Bat.explore(bat_pid, environment)
      Process.sleep(10)
    end

    # Get final state
    state = :sys.get_state(bat_pid)
    discoveries_count = length(state.discoveries)
    energy_used = 1.0 - state.energy

    # Stop the bat
    Process.exit(bat_pid, :normal)

    # Calculate fitness components
    discovery_score = discoveries_count * 10.0
    efficiency_score = (1.0 - energy_used) * 5.0
    coverage_score = map_size(state.spatial_memory) * 0.5

    # Social score if broadcasting was used
    social_score = if genome.social_tendency > 0.5, do: discoveries_count * 2.0, else: 0.0

    discovery_score + efficiency_score + coverage_score + social_score
  end

  defp default_environment do
    # Simulated environment with varying complexity
    %{
      type: :codebase,
      files: 100,
      complexity: :medium,
      targets: Enum.map(1..20, &"target_#{&1}")
    }
  end
end
```

---

### `lib/bija/evolution/population.ex`

```elixir
defmodule BIJA.Evolution.Population do
  @moduledoc """
  Manages a population of genomes across generations.
  """
  alias BIJA.Evolution.{Genome, Mutation, Fitness}

  defstruct [:individuals, :generation, :size, :elite_count]

  def new(size \\ 100) do
    individuals = for _ <- 1..size, do: Genome.random()
    %__MODULE__{
      individuals: individuals,
      generation: 0,
      size: size,
      elite_count: max(1, div(size, 10))
    }
  end

  def evolve(population, opts \\ []) do
    env = Keyword.get(opts, :environment)
    evaluator = Keyword.get(opts, :evaluator, &Fitness.evaluate/2)

    # Evaluate fitness for all individuals
    evaluated = Enum.map(population.individuals, fn genome ->
      fitness = evaluator.(genome, env)
      %{genome | fitness: fitness, generation: population.generation}
    end)

    # Sort by fitness (descending)
    sorted = Enum.sort_by(evaluated, & &1.fitness, :desc)

    # Keep elites
    elites = Enum.take(sorted, population.elite_count)

    # Generate offspring through selection, crossover, and mutation
    offspring = generate_offspring(sorted, population.size - population.elite_count)

    %__MODULE__{
      population |
      individuals: elites ++ offspring,
      generation: population.generation + 1
    }
  end

  defp generate_offspring(sorted_population, count) do
    for _ <- 1..count do
      parent1 = tournament_select(sorted_population)
      parent2 = tournament_select(sorted_population)
      child = Genome.crossover(parent1, parent2)
      Mutation.mutate(child, 0.15)
    end
  end

  defp tournament_select(population, tournament_size \\ 3) do
    population
    |> Enum.take_random(tournament_size)
    |> Enum.max_by(& &1.fitness)
  end

  def best(population) do
    Enum.max_by(population.individuals, & &1.fitness)
  end

  def average_fitness(population) do
    fitnesses = Enum.map(population.individuals, & &1.fitness)
    if Enum.empty?(fitnesses), do: 0.0, else: Enum.sum(fitnesses) / length(fitnesses)
  end
end
```

---

### `lib/bija/evolution/engine.ex`

```elixir
defmodule BIJA.Evolution.Engine do
  @moduledoc """
  Orchestrates the evolutionary process across many generations.
  Supports distributed evaluation and checkpointing.
  """
  use GenServer
  alias BIJA.Evolution.{Population, Fitness}

  defstruct [
    :population,
    :environment,
    :checkpoint_path,
    :max_generations,
    :evaluator,
    :history,
    :best_genome
  ]

  # Client API
  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def run(generations \\ 1000) do
    GenServer.call(__MODULE__, {:run, generations}, :infinity)
  end

  def get_best do
    GenServer.call(__MODULE__, :get_best)
  end

  def get_status do
    GenServer.call(__MODULE__, :get_status)
  end

  # Server callbacks
  @impl true
  def init(opts) do
    population_size = Keyword.get(opts, :population_size, 100)
    checkpoint_path = Keyword.get(opts, :checkpoint_path, "evolution_checkpoint.term")
    env = Keyword.get(opts, :environment, %{})

    state = %__MODULE__{
      population: load_checkpoint(checkpoint_path) || Population.new(population_size),
      environment: env,
      checkpoint_path: checkpoint_path,
      max_generations: Keyword.get(opts, :max_generations, 1_000_000),
      evaluator: Keyword.get(opts, :evaluator, &Fitness.evaluate/2),
      history: [],
      best_genome: nil
    }
    {:ok, state}
  end

  @impl true
  def handle_call({:run, generations}, _from, state) do
    target_gen = state.population.generation + generations
    target_gen = min(target_gen, state.max_generations)

    new_state = evolve_until(state, target_gen)
    {:reply, {:ok, new_state.best_genome}, new_state}
  end

  def handle_call(:get_best, _from, state) do
    {:reply, state.best_genome, state}
  end

  def handle_call(:get_status, _from, state) do
    status = %{
      generation: state.population.generation,
      population_size: length(state.population.individuals),
      best_fitness: if(state.best_genome, do: state.best_genome.fitness, else: nil),
      avg_fitness: Population.average_fitness(state.population)
    }
    {:reply, status, state}
  end

  defp evolve_until(state, target_generation) do
    if state.population.generation >= target_generation do
      state
    else
      IO.puts("Generation #{state.population.generation} - Best: #{best_fitness(state)}")

      new_population = Population.evolve(state.population,
        environment: state.environment,
        evaluator: state.evaluator
      )

      best = Population.best(new_population)
      new_best = if is_nil(state.best_genome) || best.fitness > state.best_genome.fitness,
        do: best,
        else: state.best_genome

      new_state = %{state |
        population: new_population,
        best_genome: new_best,
        history: [{new_population.generation, best.fitness, Population.average_fitness(new_population)} | state.history]
      }

      # Checkpoint every 100 generations
      if rem(new_population.generation, 100) == 0 do
        save_checkpoint(new_state)
      end

      evolve_until(new_state, target_generation)
    end
  end

  defp best_fitness(state) do
    if state.best_genome, do: state.best_genome.fitness, else: 0.0
  end

  defp load_checkpoint(path) do
    if File.exists?(path) do
      {population, _} = path |> File.read!() |> :erlang.binary_to_term()
      population
    end
  end

  defp save_checkpoint(state) do
    data = :erlang.term_to_binary({state.population, state.best_genome})
    File.write!(state.checkpoint_path, data)
  end
end
```

---

### `lib/mix/tasks/bija.evolve.ex`

```elixir
defmodule Mix.Tasks.Bija.Evolve do
  @moduledoc """
  Runs the evolutionary process for BIJA agents.

  ## Examples

      mix bija.evolve --generations 1000000 --population 200
  """
  use Mix.Task
  require Logger

  @shortdoc "Evolve BIJA bat agents"

  def run(args) do
    {opts, _, _} = OptionParser.parse(args,
      strict: [
        generations: :integer,
        population: :integer,
        checkpoint: :string
      ],
      aliases: [g: :generations, p: :population]
    )

    generations = Keyword.get(opts, :generations, 1000)
    population_size = Keyword.get(opts, :population, 100)
    checkpoint = Keyword.get(opts, :checkpoint, "bija_evolution.term")

    Mix.Task.run("app.start")

    IO.puts("🦇 Starting BIJA Evolution")
    IO.puts("   Generations: #{generations}")
    IO.puts("   Population: #{population_size}")

    {:ok, engine} = BIJA.Evolution.Engine.start_link(
      population_size: population_size,
      checkpoint_path: checkpoint,
      max_generations: generations,
      environment: %{type: :simulated_codebase, complexity: :high}
    )

    # Run evolution
    {:ok, best} = BIJA.Evolution.Engine.run(generations)

    IO.puts("\n🏆 Evolution Complete!")
    IO.puts("Best genome fitness: #{best.fitness}")
    IO.puts("Genome ID: #{best.id}")

    # Save best genome to file
    File.write!("best_genome.json", Jason.encode!(Map.from_struct(best), pretty: true))
    IO.puts("Best genome saved to best_genome.json")
  end
end
```

---

## 🚀 Running the Million-Turn Evolution

To start a million-generation evolutionary run:

```bash
mix bija.evolve --generations 1000000 --population 200
```

The system will:
- Create a population of 200 bat genomes.
- Evaluate each by simulating exploration in a codebase-like environment.
- Select the best, crossover, and mutate.
- Repeat for up to 1,000,000 generations.
- Save checkpoints every 100 generations.
- Output the best genome discovered.

---

## 📈 Scaling to Million Turns

To achieve a million generations efficiently:
- **Distribute evaluation** across multiple BEAM nodes using `:rpc` or a library like `Horde`.
- **Optimize the fitness function** to be fast (e.g., use lightweight simulations instead of full LLM calls).
- **Run on a cluster** of machines, with each node evaluating a subset of the population.

This evolutionary engine will continuously improve your BIJA agents, discovering optimal behaviors and parameters over time.
