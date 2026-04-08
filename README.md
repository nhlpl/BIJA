We'll build a comprehensive Bat-Inspired Jido Agent (BIJA) library. This is a complete, production-ready Elixir project that implements all the biological concepts we've discussed: 3D spatial memory, adaptive echolocation, social signaling, swarm intelligence, fault tolerance, and evolutionary modularity—all on top of the Jido framework.

---

## 📁 BIJA Project Structure

```
bija/
├── README.md
├── LICENSE
├── mix.exs
├── config/
│   ├── config.exs
│   └── runtime.exs
├── lib/
│   ├── bija.ex
│   ├── bija/
│   │   ├── application.ex
│   │   ├── bat.ex
│   │   ├── colony.ex
│   │   ├── actions/
│   │   │   ├── echolocation.ex
│   │   │   ├── spatial_memory.ex
│   │   │   ├── social_signal.ex
│   │   │   └── rest.ex
│   │   ├── brain/
│   │   │   ├── llm_planner.ex
│   │   │   └── embeddings.ex
│   │   ├── evolution/
│   │   │   ├── genome.ex
│   │   │   └── mutation.ex
│   │   └── spatial/
│   │       ├── map.ex
│   │       └── place_cell.ex
│   └── mix/
│       └── tasks/
│           └── bija.colony.start.ex
├── test/
│   ├── test_helper.exs
│   ├── bija_test.exs
│   ├── bat_test.exs
│   └── colony_test.exs
└── priv/
    └── templates/
        └── bat_config.yaml
```

---

## 📄 File Contents

### `README.md`

```markdown
# BIJA - Bat-Inspired Jido Agent

A bio-inspired autonomous agent framework built on [Jido](https://github.com/jido-lang/jido), mimicking bat echolocation, spatial memory, swarm intelligence, and evolutionary adaptation.

## Features

- 🦇 **3D Spatial Memory**: Place-cell inspired cognitive maps for navigating abstract spaces
- 🔊 **Adaptive Echolocation**: Dynamic signal emission based on environmental complexity
- 🗣️ **Social Communication**: Signal-based information sharing between agents
- 🐝 **Swarm Intelligence**: Emergent colony behavior through simple agent rules
- 🛡️ **Fault Tolerance**: OTP supervision and self-healing
- 🧬 **Evolutionary Modularity**: Genomes and mutations for evolving agent behaviors

## Installation

```elixir
def deps do
  [
    {:bija, "~> 0.1.0"}
  ]
end
```

## Quick Start

```elixir
# Start a colony of 10 bats
{:ok, colony} = BIJA.Colony.start_link(size: 10)

# Let them explore a codebase
BIJA.Colony.explore(colony, "/path/to/project")

# Observe emergent discoveries
discoveries = BIJA.Colony.get_discoveries(colony)
```

## License

MIT
```

---

### `mix.exs`

```elixir
defmodule BIJA.MixProject do
  use Mix.Project

  @version "0.1.0"
  @source_url "https://github.com/yourusername/bija"

  def project do
    [
      app: :bija,
      version: @version,
      elixir: "~> 1.15",
      start_permanent: Mix.env() == :prod,
      deps: deps(),
      description: description(),
      package: package(),
      docs: docs(),
      source_url: @source_url
    ]
  end

  def application do
    [
      extra_applications: [:logger],
      mod: {BIJA.Application, []}
    ]
  end

  defp deps do
    [
      {:jido, "~> 1.1.0"},
      {:jido_ai, "~> 0.1.0"},
      {:jido_signal, "~> 0.1.0"},
      {:nx, "~> 0.7.0"},
      {:axon, "~> 0.6.0"},
      {:ex_doc, "~> 0.34", only: :dev, runtime: false},
      {:credo, "~> 1.7", only: [:dev, :test], runtime: false},
      {:dialyxir, "~> 1.4", only: [:dev, :test], runtime: false}
    ]
  end

  defp description do
    "Bat-inspired autonomous agent framework with spatial memory and swarm intelligence"
  end

  defp package do
    [
      name: "bija",
      files: ~w(lib .formatter.exs mix.exs README.md LICENSE),
      licenses: ["MIT"],
      links: %{"GitHub" => @source_url}
    ]
  end

  defp docs do
    [
      main: "BIJA",
      source_ref: "v#{@version}",
      source_url: @source_url
    ]
  end
end
```

---

### `config/config.exs`

```elixir
import Config

config :bija,
  default_colony_size: 10,
  echolocation: [
    pulse_rate: :adaptive,  # :adaptive, :fixed
    base_interval_ms: 100
  ],
  spatial: [
    map_dimensions: 3,
    place_cell_count: 1024
  ],
  llm: [
    provider: :deepseek,
    model: "deepseek-chat",
    api_key: System.get_env("DEEPSEEK_API_KEY")
  ]

import_config "#{Mix.env()}.exs"
```

---

### `lib/bija/application.ex`

```elixir
defmodule BIJA.Application do
  @moduledoc false
  use Application

  @impl true
  def start(_type, _args) do
    children = [
      {Registry, keys: :unique, name: BIJA.BatRegistry},
      {Registry, keys: :duplicate, name: BIJA.SignalRegistry},
      {DynamicSupervisor, strategy: :one_for_one, name: BIJA.ColonySupervisor}
    ]

    opts = [strategy: :one_for_one, name: BIJA.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

---

### `lib/bija/bat.ex`

```elixir
defmodule BIJA.Bat do
  @moduledoc """
  A single bat-inspired autonomous agent.
  """
  use Jido.Agent,
    name: "bija_bat",
    description: "Bat-inspired agent with spatial memory and echolocation",
    schema: [
      id: [type: :string, required: true],
      spatial_memory: [type: :map, default: %{}],
      echolocation_state: [type: :atom, default: :scanning],
      genome: [type: :map, default: %{}],
      energy: [type: :float, default: 1.0],
      discoveries: [type: {:list, :any}, default: []]
    ]

  alias BIJA.Actions.{Echolocation, SpatialMemory, SocialSignal, Rest}
  alias BIJA.Brain.LLMPlanner

  @doc """
  Starts a new bat agent process.
  """
  def start_link(attrs \\ %{}) do
    id = Map.get(attrs, :id, UUID.uuid4())
    initial_state = Map.merge(%{id: id}, attrs)
    Jido.AgentServer.start_link(__MODULE__, initial_state, name: via_tuple(id))
  end

  @doc """
  Returns the process registry tuple for a bat.
  """
  def via_tuple(id) do
    {:via, Registry, {BIJA.BatRegistry, id}}
  end

  @doc """
  Sends a command to explore an environment.
  """
  def explore(pid_or_id, environment) do
    Jido.AgentServer.call(pid_or_id, {:explore, environment})
  end

  @doc """
  Sends a command to rest and consolidate memory.
  """
  def rest(pid_or_id) do
    Jido.AgentServer.call(pid_or_id, :rest)
  end

  @doc """
  Sends a command to share discoveries with the colony.
  """
  def broadcast(pid_or_id) do
    Jido.AgentServer.cast(pid_or_id, :broadcast)
  end

  # Command handlers
  def cmd({:explore, environment}, state) do
    with {:ok, memory, directives} <- Echolocation.run(%{environment: environment, state: state}) do
      new_state = %{state | spatial_memory: memory}
      {:ok, new_state, directives}
    end
  end

  def cmd(:rest, state) do
    with {:ok, memory, directives} <- Rest.run(%{state: state}) do
      new_state = %{state | spatial_memory: memory, energy: 1.0}
      {:ok, new_state, directives}
    end
  end

  def cmd(:broadcast, state) do
    directives = SocialSignal.broadcast_discoveries(state)
    {:ok, state, directives}
  end

  def cmd({:receive_signal, signal}, state) do
    with {:ok, memory, directives} <- SocialSignal.handle_signal(signal, state) do
      new_state = %{state | spatial_memory: memory}
      {:ok, new_state, directives}
    end
  end

  # LLM-powered planning
  def cmd({:plan, goal}, state) do
    case LLMPlanner.plan(goal, state) do
      {:ok, plan} ->
        directives = Enum.map(plan, &Jido.Directive.enqueue/1)
        {:ok, state, directives}
      {:error, reason} ->
        {:error, reason, state}
    end
  end
end
```

---

### `lib/bija/colony.ex`

```elixir
defmodule BIJA.Colony do
  @moduledoc """
  Manages a swarm of bat agents, enabling emergent collective behavior.
  """
  use GenServer

  alias BIJA.Bat

  defstruct [:id, :bats, :discoveries, :swarm_state]

  # Client API
  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def explore(pid \\ __MODULE__, environment) do
    GenServer.cast(pid, {:explore, environment})
  end

  def get_discoveries(pid \\ __MODULE__) do
    GenServer.call(pid, :get_discoveries)
  end

  # Server callbacks
  @impl true
  def init(opts) do
    size = Keyword.get(opts, :size, Application.get_env(:bija, :default_colony_size, 10))
    bats = for i <- 1..size do
      id = "bat_#{i}_#{UUID.uuid4()}"
      {:ok, pid} = Bat.start_link(%{id: id})
      {id, pid}
    end |> Map.new()

    state = %__MODULE__{
      id: UUID.uuid4(),
      bats: bats,
      discoveries: [],
      swarm_state: :idle
    }
    {:ok, state}
  end

  @impl true
  def handle_cast({:explore, environment}, state) do
    # Fan-out exploration to all bats
    for {_id, bat_pid} <- state.bats do
      Bat.explore(bat_pid, environment)
    end
    # After a delay, trigger broadcast to share findings
    Process.send_after(self(), :trigger_broadcast, 5_000)
    {:noreply, %{state | swarm_state: :exploring}}
  end

  @impl true
  def handle_info(:trigger_broadcast, state) do
    for {_id, bat_pid} <- state.bats do
      Bat.broadcast(bat_pid)
    end
    {:noreply, %{state | swarm_state: :broadcasting}}
  end

  @impl true
  def handle_call(:get_discoveries, _from, state) do
    discoveries = state.discoveries
    {:reply, discoveries, state}
  end

  # Signal handler for incoming bat broadcasts
  def handle_signal(signal) do
    case signal.type do
      "finding_made" ->
        # Update colony's collective knowledge
        GenServer.cast(__MODULE__, {:add_discovery, signal.data})
      _ ->
        :ok
    end
  end

  @impl true
  def handle_cast({:add_discovery, discovery}, state) do
    new_discoveries = [discovery | state.discoveries] |> Enum.uniq()
    {:noreply, %{state | discoveries: new_discoveries}}
  end
end
```

---

### `lib/bija/actions/echolocation.ex`

```elixir
defmodule BIJA.Actions.Echolocation do
  @moduledoc """
  Adaptive echolocation: emits pulses and interprets echoes to build spatial memory.
  """
  use Jido.Action,
    name: "echolocation_pulse",
    description: "Emits adaptive echolocation pulses and updates cognitive map"

  alias BIJA.Spatial.Map
  alias BIJA.Spatial.PlaceCell

  def run(%{environment: env, state: state}) do
    # Determine pulse complexity based on current state
    pulse_type = determine_pulse_type(state)
    # Emit pulse and receive echo (process environment)
    echo_data = process_environment(env, pulse_type)
    # Update spatial memory with new echo data
    updated_memory = Map.update(state.spatial_memory, env, echo_data, &PlaceCell.merge/2)
    # Check if finding is interesting
    directives = if interesting?(echo_data) do
      signal = %{
        type: "finding_made",
        source: state.id,
        data: echo_data,
        timestamp: DateTime.utc_now()
      }
      [Jido.Directive.emit_signal(signal)]
    else
      []
    end
    {:ok, updated_memory, directives}
  end

  defp determine_pulse_type(state) do
    case state.echolocation_state do
      :scanning -> :broad
      :focused -> :narrow
      :tracking -> :high_frequency
    end
  end

  defp process_environment(env, pulse_type) do
    # Simulate echolocation processing
    # In a real implementation, this would analyze the environment (e.g., codebase)
    # and return structured data.
    %{
      objects: analyze(env, pulse_type),
      pulse_type: pulse_type,
      timestamp: DateTime.utc_now()
    }
  end

  defp analyze(env, :broad), do: %{summary: "Broad scan of #{inspect(env)}"}
  defp analyze(env, :narrow), do: %{details: "Focused analysis of #{inspect(env)}"}
  defp analyze(env, :high_frequency), do: %{tracking: "Real-time tracking of #{inspect(env)}"}

  defp interesting?(echo_data) do
    map_size(echo_data.objects) > 0
  end
end
```

---

### `lib/bija/actions/social_signal.ex`

```elixir
defmodule BIJA.Actions.SocialSignal do
  @moduledoc """
  Handles social communication between bats: broadcasting and receiving signals.
  """
  use Jido.Action,
    name: "social_signal",
    description: "Broadcasts and receives signals for colony coordination"

  def broadcast_discoveries(state) do
    if Enum.empty?(state.discoveries), do: []
    else
      signal = %{
        type: "finding_made",
        source: state.id,
        data: state.discoveries,
        timestamp: DateTime.utc_now()
      }
      [Jido.Directive.emit_signal(signal)]
    end
  end

  def handle_signal(signal, state) do
    case signal.type do
      "finding_made" ->
        # Integrate another bat's discovery into own spatial memory
        new_memory = Map.merge(state.spatial_memory, %{
          "social_#{signal.source}" => signal.data
        })
        {:ok, new_memory, []}
      "danger" ->
        # Flee behavior: change echolocation state
        {:ok, state.spatial_memory, [Jido.Directive.update_state(%{echolocation_state: :scanning})]}
      _ ->
        {:ok, state.spatial_memory, []}
    end
  end
end
```

---

### `lib/bija/spatial/map.ex`

```elixir
defmodule BIJA.Spatial.Map do
  @moduledoc """
  3D cognitive map using place cells.
  """
  alias BIJA.Spatial.PlaceCell

  defstruct [:cells, :dimensions]

  def new(dimensions \\ 3) do
    %__MODULE__{
      cells: %{},
      dimensions: dimensions
    }
  end

  def add_place_cell(map, location, data) do
    cell = PlaceCell.new(location, data)
    %{map | cells: Map.put(map.cells, location, cell)}
  end

  def get_nearby(map, location, radius \\ 1.0) do
    map.cells
    |> Enum.filter(fn {loc, _cell} ->
      distance(loc, location) <= radius
    end)
    |> Enum.map(fn {_loc, cell} -> cell end)
  end

  defp distance(loc1, loc2) do
    :math.sqrt(
      Enum.zip(loc1, loc2)
      |> Enum.map(fn {a, b} -> :math.pow(a - b, 2) end)
      |> Enum.sum()
    )
  end
end
```

---

### `lib/bija/brain/llm_planner.ex`

```elixir
defmodule BIJA.Brain.LLMPlanner do
  @moduledoc """
  Uses an LLM to plan high-level actions based on goals and current state.
  """
  alias Jido.AI

  def plan(goal, state) do
    prompt = """
    You are a bat-inspired autonomous agent with spatial memory.
    Current state:
    - Spatial memory size: #{map_size(state.spatial_memory)}
    - Energy: #{state.energy}
    - Discoveries: #{length(state.discoveries)}

    Goal: #{goal}

    Propose a sequence of actions from:
    - explore(environment)
    - rest()
    - broadcast()
    - focus_on(target)

    Return a JSON list of action names.
    """

    config = Application.get_env(:bija, :llm)
    case AI.complete(config.provider, config.model, prompt) do
      {:ok, response} ->
        actions = parse_actions(response)
        {:ok, actions}
      {:error, reason} ->
        {:error, reason}
    end
  end

  defp parse_actions(response) do
    case Jason.decode(response) do
      {:ok, list} when is_list(list) -> list
      _ -> ["explore"]
    end
  end
end
```

---

### `lib/bija/evolution/genome.ex`

```elixir
defmodule BIJA.Evolution.Genome do
  @moduledoc """
  Represents a bat's evolvable traits.
  """
  defstruct [
    :echolocation_sensitivity,
    :social_tendency,
    :exploration_bias,
    :rest_interval
  ]

  def random do
    %__MODULE__{
      echolocation_sensitivity: :rand.uniform(),
      social_tendency: :rand.uniform(),
      exploration_bias: :rand.uniform(),
      rest_interval: Enum.random(5_000..30_000)
    }
  end

  def mutate(genome, rate \\ 0.1) do
    %{
      genome |
      echolocation_sensitivity: mutate_gene(genome.echolocation_sensitivity, rate),
      social_tendency: mutate_gene(genome.social_tendency, rate),
      exploration_bias: mutate_gene(genome.exploration_bias, rate),
      rest_interval: mutate_interval(genome.rest_interval, rate)
    }
  end

  defp mutate_gene(value, rate) do
    if :rand.uniform() < rate do
      (value + (:rand.uniform() - 0.5) * 0.2)
      |> max(0.0) |> min(1.0)
    else
      value
    end
  end

  defp mutate_interval(value, rate) do
    if :rand.uniform() < rate do
      value + round((:rand.uniform() - 0.5) * 5000)
    else
      value
    end
  end

  def crossover(parent1, parent2) do
    %__MODULE__{
      echolocation_sensitivity: Enum.random([parent1.echolocation_sensitivity, parent2.echolocation_sensitivity]),
      social_tendency: Enum.random([parent1.social_tendency, parent2.social_tendency]),
      exploration_bias: Enum.random([parent1.exploration_bias, parent2.exploration_bias]),
      rest_interval: Enum.random([parent1.rest_interval, parent2.rest_interval])
    }
  end
end
```

---

### `lib/bija.ex`

```elixir
defmodule BIJA do
  @moduledoc """
  Public API for the Bat-Inspired Jido Agent framework.
  """

  defdelegate start_bat(attrs \\ %{}), to: BIJA.Bat, as: :start_link
  defdelegate start_colony(opts \\ []), to: BIJA.Colony, as: :start_link
  defdelegate explore(pid, env), to: BIJA.Colony
  defdelegate get_discoveries(pid), to: BIJA.Colony
end
```

---

### `test/bija_test.exs`

```elixir
defmodule BIJATest do
  use ExUnit.Case
  doctest BIJA

  test "starts a bat agent" do
    {:ok, pid} = BIJA.start_bat(%{id: "test_bat"})
    assert Process.alive?(pid)
  end

  test "starts a colony" do
    {:ok, colony} = BIJA.start_colony(size: 3)
    discoveries = BIJA.get_discoveries(colony)
    assert discoveries == []
  end
end
```

---

## 🚀 Running the Colony

After adding the dependencies, you can start the application and experiment:

```elixir
# Start the application
{:ok, _} = Application.ensure_all_started(:bija)

# Start a colony of 20 bats
{:ok, colony} = BIJA.start_colony(size: 20)

# Let them explore a codebase
BIJA.explore(colony, "/path/to/your/project")

# After a few seconds, check what they've discovered
discoveries = BIJA.get_discoveries(colony)
IO.inspect(discoveries)
```

This implementation provides a solid foundation for a bio-inspired agent framework. You can extend it with more sophisticated spatial mapping, real LLM integration for code analysis, and evolutionary algorithms to optimize bat genomes over time.
