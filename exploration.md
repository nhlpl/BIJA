We'll enhance the evolved BIJA agent with mathematically rigorous observation models. Instead of heuristic rules, the bat will use Bayesian inference, information theory, and probabilistic spatial mapping to make optimal decisions based on sensory data.

---

## 🧮 Mathematical Observation Modules

### `lib/bija/math/observation_model.ex`

```elixir
defmodule BIJA.Math.ObservationModel do
  @moduledoc """
  Bayesian observation model for echolocation returns.
  
  Models the environment as a set of latent features with associated probabilities.
  Each echolocation pulse returns a noisy observation vector.
  """

  alias __MODULE__

  defstruct [
    :feature_priors,      # %{feature_id => prior_probability}
    :observation_matrix,   # %{feature_id => %{observation => likelihood}}
    :current_beliefs       # %{feature_id => posterior_probability}
  ]

  @doc """
  Initialize model with uniform priors over possible features.
  """
  def new(features, observation_space) do
    n = length(features)
    priors = Map.new(features, &{&1, 1.0 / n})
    obs_matrix = build_observation_matrix(features, observation_space)
    
    %__MODULE__{
      feature_priors: priors,
      observation_matrix: obs_matrix,
      current_beliefs: priors
    }
  end

  defp build_observation_matrix(features, obs_space) do
    # Build a simple generative model: each feature produces observations 
    # with probability proportional to cosine similarity of embeddings
    # In practice, this would be learned or defined by domain knowledge
    Map.new(features, fn feature ->
      probs = Map.new(obs_space, fn obs ->
        {obs, observation_likelihood(feature, obs)}
      end)
      {feature, probs}
    end)
  end

  defp observation_likelihood(feature, observation) do
    # Placeholder: use embedding similarity if available
    1.0 / (1.0 + abs(:erlang.phash2(feature) - :erlang.phash2(observation)) / 1_000_000)
  end

  @doc """
  Bayesian update: P(feature | observation) ∝ P(observation | feature) * P(feature)
  """
  def update(model, observation) do
    new_beliefs = Map.new(model.current_beliefs, fn {feature, prior} ->
      likelihood = get_in(model.observation_matrix, [feature, observation]) || 0.01
      {feature, likelihood * prior}
    end)
    
    # Normalize
    total = Enum.sum(Map.values(new_beliefs))
    normalized = if total > 0 do
      Map.new(new_beliefs, fn {k, v} -> {k, v / total} end)
    else
      model.feature_priors
    end
    
    %{model | current_beliefs: normalized}
  end

  @doc """
  Compute entropy of current beliefs: H = -Σ p log p
  """
  def entropy(model) do
    model.current_beliefs
    |> Map.values()
    |> Enum.reduce(0.0, fn p, acc ->
      if p > 0, do: acc - p * :math.log2(p), else: acc
    end)
  end

  @doc """
  Most likely feature given current beliefs.
  """
  def map_estimate(model) do
    {feature, _prob} = Enum.max_by(model.current_beliefs, fn {_, p} -> p end)
    feature
  end

  @doc """
  Sample a feature according to posterior probability.
  """
  def sample_feature(model) do
    r = :rand.uniform()
    model.current_beliefs
    |> Enum.reduce({r, nil}, fn {f, p}, {remaining, _} ->
      if remaining <= p and is_nil(elem({remaining, p}, 1)) do
        {remaining, f}
      else
        {remaining - p, nil}
      end
    end)
    |> elem(1)
  end
end
```

---

### `lib/bija/math/information_gain.ex`

```elixir
defmodule BIJA.Math.InformationGain do
  @moduledoc """
  Compute expected information gain for candidate actions.
  
  Used to guide exploration: choose action that maximizes reduction in uncertainty.
  """

  alias BIJA.Math.ObservationModel

  @doc """
  Expected information gain of making an observation.
  
  EIG = H(current) - E[H(posterior)]
  where expectation is over possible observations.
  """
  def expected_information_gain(model, possible_observations) do
    current_entropy = ObservationModel.entropy(model)
    
    expected_posterior_entropy = Enum.reduce(possible_observations, 0.0, fn obs, acc ->
      prob_obs = predictive_probability(model, obs)
      if prob_obs > 0 do
        posterior = ObservationModel.update(model, obs)
        posterior_entropy = ObservationModel.entropy(posterior)
        acc + prob_obs * posterior_entropy
      else
        acc
      end
    end)
    
    current_entropy - expected_posterior_entropy
  end

  defp predictive_probability(model, observation) do
    Enum.reduce(model.current_beliefs, 0.0, fn {feature, prior}, acc ->
      likelihood = get_in(model.observation_matrix, [feature, observation]) || 0.0
      acc + prior * likelihood
    end)
  end

  @doc """
  Select the observation that maximizes expected information gain.
  """
  def select_most_informative_observation(model, candidate_observations) do
    candidate_observations
    |> Enum.map(fn obs -> {obs, expected_information_gain(model, [obs])} end)
    |> Enum.max_by(fn {_, eig} -> eig end)
    |> elem(0)
  end
end
```

---

### `lib/bija/math/spatial_inference.ex`

```elixir
defmodule BIJA.Math.SpatialInference do
  @moduledoc """
  Probabilistic 3D spatial mapping using place cells with uncertainty.
  
  Each place cell has a preferred location (center) and a Gaussian tuning curve.
  Given an observation, we compute posterior over locations.
  """

  defstruct [
    :place_cells,           # list of %{id: term, center: [x,y,z], width: float}
    :location_prior,        # %{location_hash => probability}
    :current_location_belief
  ]

  alias __MODULE__

  @doc """
  Initialize with a grid of place cells covering the space.
  """
  def new(bounds \\ {[-10, -10, -10], [10, 10, 10]}, resolution \\ 1.0) do
    {min_coords, max_coords} = bounds
    cells = generate_grid_place_cells(min_coords, max_coords, resolution)
    
    uniform = 1.0 / length(cells)
    priors = Map.new(cells, &{&1.id, uniform})
    
    %__MODULE__{
      place_cells: cells,
      location_prior: priors,
      current_location_belief: priors
    }
  end

  defp generate_grid_place_cells(min, max, res) do
    for x <- Stream.iterate(min |> elem(0), &(&1 + res)) |> Enum.take_while(&(&1 <= elem(max, 0))),
        y <- Stream.iterate(min |> elem(1), &(&1 + res)) |> Enum.take_while(&(&1 <= elem(max, 1))),
        z <- Stream.iterate(min |> elem(2), &(&1 + res)) |> Enum.take_while(&(&1 <= elem(max, 2))) do
      %{
        id: "cell_#{x}_#{y}_#{z}",
        center: [x, y, z],
        width: res * 2.0
      }
    end
  end

  @doc """
  Likelihood of observation given location: P(obs | loc) ∝ exp(-distance^2 / 2*width^2)
  """
  def observation_likelihood(observation_features, place_cell) do
    # Convert observation to a pseudo-location based on features
    obs_loc = features_to_location(observation_features)
    dist = euclidean_distance(obs_loc, place_cell.center)
    variance = place_cell.width ** 2
    :math.exp(-dist ** 2 / (2 * variance))
  end

  defp features_to_location(features) do
    # Hash features to a 3D coordinate (simplified)
    hash = :erlang.phash2(features, 1_000_000)
    x = rem(hash, 100) / 10.0 - 5.0
    y = rem(div(hash, 100), 100) / 10.0 - 5.0
    z = rem(div(hash, 10_000), 100) / 10.0 - 5.0
    [x, y, z]
  end

  defp euclidean_distance(a, b) do
    :math.sqrt(Enum.zip(a, b) |> Enum.map(fn {x, y} -> (x - y) ** 2 end) |> Enum.sum())
  end

  @doc """
  Bayesian update of location belief given observation.
  """
  def update(model, observation_features) do
    new_beliefs = Map.new(model.current_location_belief, fn {cell_id, prior} ->
      cell = Enum.find(model.place_cells, &(&1.id == cell_id))
      likelihood = observation_likelihood(observation_features, cell)
      {cell_id, likelihood * prior}
    end)
    
    # Normalize
    total = Enum.sum(Map.values(new_beliefs))
    normalized = if total > 0 do
      Map.new(new_beliefs, fn {k, v} -> {k, v / total} end)
    else
      model.location_prior
    end
    
    %{model | current_location_belief: normalized}
  end

  @doc """
  Sample a location from the posterior.
  """
  def sample_location(model) do
    r = :rand.uniform()
    {_, cell_id} = Enum.reduce_while(model.current_location_belief, {r, nil}, fn {id, p}, {rem, _} ->
      if rem <= p do
        {:halt, {rem, id}}
      else
        {:cont, {rem - p, nil}}
      end
    end)
    cell = Enum.find(model.place_cells, &(&1.id == cell_id))
    cell.center
  end

  @doc """
  Compute spatial entropy (uncertainty about location).
  """
  def entropy(model) do
    model.current_location_belief
    |> Map.values()
    |> Enum.reduce(0.0, fn p, acc ->
      if p > 0, do: acc - p * :math.log2(p), else: acc
    end)
  end
end
```

---

### `lib/bija/math/adaptive_sampling.ex`

```elixir
defmodule BIJA.Math.AdaptiveSampling do
  @moduledoc """
  Optimal pulse scheduling using Poisson process models and information gain.
  
  Determines when to emit the next echolocation pulse based on expected
  information gain and current uncertainty.
  """

  alias BIJA.Math.{ObservationModel, InformationGain, SpatialInference}

  @doc """
  Compute optimal inter-pulse interval based on current uncertainty.
  
  Higher uncertainty → shorter interval (more frequent pulses).
  Lower uncertainty → longer interval (energy conservation).
  """
  def optimal_interval(model, spatial_model, base_interval \\ 45, max_interval \\ 230) do
    # Combine entropies from both models
    feature_entropy = ObservationModel.entropy(model)
    spatial_entropy = SpatialInference.entropy(spatial_model)
    
    # Normalized uncertainty (0 to 1)
    max_possible_entropy = :math.log2(length(Map.keys(model.feature_priors)))
    uncertainty = (feature_entropy + spatial_entropy) / (2 * max_possible_entropy)
    uncertainty = min(1.0, max(0.0, uncertainty))
    
    # Map uncertainty to interval: high uncertainty -> short interval
    round(base_interval + (max_interval - base_interval) * (1 - uncertainty))
  end

  @doc """
  Decide whether to emit a pulse now based on information gain threshold.
  """
  def should_pulse?(model, spatial_model, possible_observations, threshold \\ 0.1) do
    eig = InformationGain.expected_information_gain(model, possible_observations)
    spatial_entropy = SpatialInference.entropy(spatial_model)
    
    # Pulse if expected info gain is high or spatial uncertainty is high
    eig > threshold or spatial_entropy > 2.0
  end

  @doc """
  Model the observation process as a Poisson process with rate λ.
  λ is proportional to uncertainty.
  """
  def sample_next_pulse_time(model, spatial_model, base_rate \\ 0.02) do
    uncertainty = (ObservationModel.entropy(model) + SpatialInference.entropy(spatial_model)) / 10.0
    rate = base_rate * (1 + uncertainty)
    # Exponential distribution: -ln(U) / λ
    -:math.log(:rand.uniform()) / rate
  end
end
```

---

### Integration: Updated `echolocation.ex` with Math Models

```elixir
defmodule BIJA.Actions.Echolocation do
  @moduledoc """
  Evolved adaptive echolocation using Bayesian observation models.
  """
  use Jido.Action,
    name: "echolocation_pulse_v3",
    description: "Mathematically optimal echolocation using Bayesian inference"

  alias BIJA.Math.{ObservationModel, InformationGain, SpatialInference, AdaptiveSampling}
  alias BIJA.Spatial.Map

  def run(%{environment: env, state: state}) do
    # Initialize or retrieve mathematical models from state
    obs_model = get_obs_model(state)
    spatial_model = get_spatial_model(state)
    
    # Determine candidate observations (possible things to "look" for)
    candidates = generate_candidate_observations(env)
    
    # Decide whether to pulse based on information gain
    if AdaptiveSampling.should_pulse?(obs_model, spatial_model, candidates, 0.05) do
      # Choose most informative observation to focus on
      target_obs = InformationGain.select_most_informative_observation(obs_model, candidates)
      
      # Emit pulse and receive echo
      echo_data = process_environment(env, target_obs)
      
      # Bayesian update
      new_obs_model = ObservationModel.update(obs_model, echo_data.observation)
      new_spatial_model = SpatialInference.update(spatial_model, echo_data.features)
      
      # Update state with new models
      updated_memory = store_in_spatial_memory(state.spatial_memory, echo_data)
      
      directives = if interesting?(new_obs_model, echo_data) do
        signal = %{
          type: "finding_made",
          source: state.id,
          data: echo_data,
          confidence: 1.0 - ObservationModel.entropy(new_obs_model) / 10.0,
          timestamp: DateTime.utc_now()
        }
        [Jido.Directive.emit_signal(signal)]
      else
        []
      end
      
      # Store models in state for next pulse
      new_state = state
        |> Map.put(:obs_model, new_obs_model)
        |> Map.put(:spatial_model, new_spatial_model)
        |> Map.put(:spatial_memory, updated_memory)
        |> Map.put(:last_pulse_time, System.monotonic_time(:millisecond))
      
      {:ok, updated_memory, directives, new_state}
    else
      # Skip pulse to conserve energy
      {:ok, state.spatial_memory, [], state}
    end
  end

  defp get_obs_model(state) do
    Map.get(state, :obs_model) || 
      ObservationModel.new(["code", "config", "doc", "test", "data"], ~w(structured unstructured complex simple))
  end

  defp get_spatial_model(state) do
    Map.get(state, :spatial_model) || SpatialInference.new()
  end

  defp generate_candidate_observations(env) do
    # Extract possible things to observe from environment
    case env[:type] do
      :codebase -> ["function", "module", "dependency", "pattern"]
      _ -> Map.keys(env)
    end
  end

  defp process_environment(env, target) do
    # Simulate echolocation return
    observation = if env[:type] == :codebase do
      Enum.random(["function", "module", "dependency", "pattern"])
    else
      target
    end
    
    features = %{
      complexity: env[:complexity] || 0.5,
      size: map_size(env),
      target: target
    }
    
    %{
      observation: observation,
      features: features,
      timestamp: DateTime.utc_now(),
      signature: :crypto.hash(:sha256, inspect({env, target})) |> Base.encode16()
    }
  end

  defp interesting?(obs_model, echo_data) do
    entropy = ObservationModel.entropy(obs_model)
    max_entropy = :math.log2(map_size(obs_model.feature_priors))
    confidence = 1.0 - entropy / max_entropy
    confidence > 0.7
  end

  defp store_in_spatial_memory(memory, echo_data) do
    Map.put(memory, echo_data.signature, echo_data)
  end
end
```

---

### Updated `bat.ex` to Include Math State

Add these fields to the schema:

```elixir
schema: [
  # ... existing fields ...
  obs_model: [type: :any, default: nil],
  spatial_model: [type: :any, default: nil]
]
```

And update the cmd/2 for explore to pass the state through:

```elixir
def cmd({:explore, environment}, state) do
  if state.energy > 0.1 do
    case Echolocation.run(%{environment: environment, state: state}) do
      {:ok, memory, directives, new_state} ->
        {:ok, new_state, directives}
      {:ok, memory, directives} ->
        new_state = %{state | spatial_memory: memory, energy: state.energy - 0.015}
        {:ok, new_state, directives}
    end
  else
    {:ok, state, [Jido.Directive.enqueue({:rest, []})]}
  end
end
```

---

## 🎯 Summary

The enhanced BIJA agent now uses:

- **Bayesian inference** to maintain probabilistic beliefs about its environment.
- **Information gain** to select the most informative observations.
- **Probabilistic spatial mapping** with place cells and Gaussian tuning curves.
- **Adaptive sampling** based on Poisson process models to optimize pulse timing.

This transforms the agent from heuristic-driven to mathematically principled, enabling optimal exploration under uncertainty while conserving energy.
