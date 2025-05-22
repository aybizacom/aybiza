# AYBIZA Advanced Analytics & Machine Learning Implementation

## Overview
This document provides comprehensive implementation guidance for advanced analytics and machine learning capabilities in the AYBIZA platform using TimescaleDB, PostgreSQL, and modern ML techniques for real-time voice analytics.

## Architecture Overview

### Analytics Stack
```
Voice Calls → Real-time Stream Processing → TimescaleDB → 
  Analytics Engine → ML Models → Business Intelligence → Dashboards
```

### Key Components
- **TimescaleDB 2.20.0+**: Hybrid row-columnar store for time-series analytics
- **PostgreSQL 16.9+**: Advanced SQL analytics with ML extensions
- **Elixir Analytics Engine**: Real-time stream processing
- **Vector Search**: pgvector + pgvectorscale for embeddings
- **ML Pipeline**: Python/Elixir hybrid for model training and inference

## Database Schema for Analytics

### 1. Time-Series Tables with TimescaleDB
```sql
-- Core call analytics table
CREATE TABLE call_analytics (
    time TIMESTAMPTZ NOT NULL,
    call_id UUID NOT NULL,
    agent_id UUID NOT NULL,
    phone_number VARCHAR(20),
    caller_number VARCHAR(20),
    region VARCHAR(20),
    
    -- Performance metrics
    total_duration_ms INTEGER,
    voice_processing_latency_ms INTEGER,
    stt_latency_ms INTEGER,
    llm_latency_ms INTEGER,
    tts_latency_ms INTEGER,
    
    -- Quality metrics
    audio_quality_score DECIMAL(3,2),
    conversation_quality_score DECIMAL(3,2),
    stt_confidence_avg DECIMAL(3,2),
    
    -- Business metrics
    call_outcome VARCHAR(50),
    customer_satisfaction_score DECIMAL(3,2),
    resolution_status VARCHAR(50),
    cost_usd DECIMAL(10,4),
    
    -- System metrics
    model_used VARCHAR(100),
    tokens_consumed INTEGER,
    error_count INTEGER,
    
    PRIMARY KEY (time, call_id)
);

-- Convert to hypertable for time-series optimization
SELECT create_hypertable('call_analytics', 'time', chunk_time_interval => INTERVAL '1 day');

-- Create compression policy for older data
ALTER TABLE call_analytics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'agent_id, region',
    timescaledb.compress_orderby = 'time'
);

SELECT add_compression_policy('call_analytics', INTERVAL '7 days');

-- Conversation turns for detailed analysis
CREATE TABLE conversation_turns (
    time TIMESTAMPTZ NOT NULL,
    call_id UUID NOT NULL,
    turn_number INTEGER NOT NULL,
    speaker VARCHAR(10) NOT NULL, -- 'caller' or 'agent'
    
    -- Content analysis
    text_content TEXT,
    sentiment_score DECIMAL(3,2),
    emotion VARCHAR(50),
    intent VARCHAR(100),
    entities JSONB,
    
    -- Performance metrics
    response_time_ms INTEGER,
    confidence_score DECIMAL(3,2),
    
    -- ML features
    text_embedding VECTOR(1536), -- OpenAI embeddings
    topic_vector VECTOR(100),    -- Custom topic embeddings
    
    PRIMARY KEY (time, call_id, turn_number)
);

SELECT create_hypertable('conversation_turns', 'time', chunk_time_interval => INTERVAL '1 day');

-- Agent performance metrics
CREATE TABLE agent_performance_metrics (
    time TIMESTAMPTZ NOT NULL,
    agent_id UUID NOT NULL,
    
    -- Aggregated metrics (hourly)
    total_calls INTEGER,
    successful_calls INTEGER,
    avg_duration_seconds DECIMAL(8,2),
    avg_satisfaction_score DECIMAL(3,2),
    avg_resolution_rate DECIMAL(3,2),
    
    -- Performance indicators
    avg_response_time_ms DECIMAL(8,2),
    avg_audio_quality DECIMAL(3,2),
    error_rate DECIMAL(5,4),
    
    -- Cost metrics
    total_cost_usd DECIMAL(10,4),
    cost_per_call_usd DECIMAL(6,4),
    
    -- Model usage
    model_distribution JSONB,
    token_consumption INTEGER,
    
    PRIMARY KEY (time, agent_id)
);

SELECT create_hypertable('agent_performance_metrics', 'time', chunk_time_interval => INTERVAL '1 day');

-- System performance metrics
CREATE TABLE system_performance_metrics (
    time TIMESTAMPTZ NOT NULL,
    region VARCHAR(20) NOT NULL,
    
    -- Infrastructure metrics
    concurrent_calls INTEGER,
    cpu_utilization_pct DECIMAL(5,2),
    memory_utilization_pct DECIMAL(5,2),
    network_latency_ms DECIMAL(8,2),
    
    -- Service health
    error_rate DECIMAL(5,4),
    availability_pct DECIMAL(5,2),
    sla_compliance_pct DECIMAL(5,2),
    
    -- Capacity metrics
    max_capacity INTEGER,
    utilization_pct DECIMAL(5,2),
    queue_depth INTEGER,
    
    PRIMARY KEY (time, region)
);

SELECT create_hypertable('system_performance_metrics', 'time', chunk_time_interval => INTERVAL '1 hour');
```

### 2. Continuous Aggregates for Real-time Analytics
```sql
-- Hourly call analytics aggregation
CREATE MATERIALIZED VIEW call_analytics_hourly
WITH (timescaledb.continuous) AS
SELECT 
    time_bucket('1 hour', time) AS bucket,
    agent_id,
    region,
    COUNT(*) AS total_calls,
    AVG(total_duration_ms) AS avg_duration_ms,
    AVG(voice_processing_latency_ms) AS avg_latency_ms,
    AVG(customer_satisfaction_score) AS avg_satisfaction,
    AVG(audio_quality_score) AS avg_audio_quality,
    SUM(cost_usd) AS total_cost_usd,
    AVG(cost_usd) AS avg_cost_per_call,
    COUNT(*) FILTER (WHERE call_outcome = 'successful') AS successful_calls,
    COUNT(*) FILTER (WHERE error_count > 0) AS calls_with_errors
FROM call_analytics
GROUP BY bucket, agent_id, region;

-- Daily business metrics
CREATE MATERIALIZED VIEW business_metrics_daily
WITH (timescaledb.continuous) AS
SELECT 
    time_bucket('1 day', time) AS bucket,
    COUNT(*) AS total_calls,
    AVG(customer_satisfaction_score) AS avg_satisfaction,
    COUNT(*) FILTER (WHERE resolution_status = 'resolved') AS resolved_calls,
    COUNT(*) FILTER (WHERE call_outcome = 'successful') AS successful_calls,
    SUM(cost_usd) AS total_revenue,
    AVG(total_duration_ms) / 1000.0 AS avg_duration_seconds,
    
    -- Performance percentiles
    percentile_cont(0.5) WITHIN GROUP (ORDER BY voice_processing_latency_ms) AS p50_latency,
    percentile_cont(0.95) WITHIN GROUP (ORDER BY voice_processing_latency_ms) AS p95_latency,
    percentile_cont(0.99) WITHIN GROUP (ORDER BY voice_processing_latency_ms) AS p99_latency
FROM call_analytics
GROUP BY bucket;

-- Agent performance ranking
CREATE MATERIALIZED VIEW agent_performance_ranking_daily
WITH (timescaledb.continuous) AS
SELECT 
    time_bucket('1 day', time) AS bucket,
    agent_id,
    COUNT(*) AS total_calls,
    AVG(customer_satisfaction_score) AS avg_satisfaction,
    AVG(total_duration_ms) / 1000.0 AS avg_duration_seconds,
    COUNT(*) FILTER (WHERE resolution_status = 'resolved')::FLOAT / COUNT(*) AS resolution_rate,
    RANK() OVER (PARTITION BY time_bucket('1 day', time) ORDER BY AVG(customer_satisfaction_score) DESC) AS satisfaction_rank,
    RANK() OVER (PARTITION BY time_bucket('1 day', time) ORDER BY COUNT(*) DESC) AS volume_rank
FROM call_analytics
GROUP BY bucket, agent_id;

-- Refresh policies for continuous aggregates
SELECT add_continuous_aggregate_policy('call_analytics_hourly',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

SELECT add_continuous_aggregate_policy('business_metrics_daily',
    start_offset => INTERVAL '2 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 hour');
```

## Machine Learning Implementation

### 1. Real-time Sentiment Analysis
```elixir
defmodule Aybiza.Analytics.SentimentAnalyzer do
  use GenServer
  require Logger

  alias Aybiza.Analytics.MLPipeline

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def init(_opts) do
    # Load pre-trained sentiment model
    {:ok, model} = MLPipeline.load_model("sentiment_analysis_v2")
    
    state = %{
      model: model,
      batch_buffer: [],
      batch_size: 50,
      last_batch_time: System.monotonic_time(:millisecond)
    }

    # Schedule periodic batch processing
    :timer.send_interval(5000, :process_batch)

    {:ok, state}
  end

  def analyze_text(text) when is_binary(text) do
    GenServer.call(__MODULE__, {:analyze_text, text})
  end

  def analyze_conversation_turn(call_id, turn_number, text, metadata \\ %{}) do
    GenServer.cast(__MODULE__, {:analyze_turn, call_id, turn_number, text, metadata})
  end

  def handle_call({:analyze_text, text}, _from, state) do
    result = perform_sentiment_analysis(text, state.model)
    {:reply, result, state}
  end

  def handle_cast({:analyze_turn, call_id, turn_number, text, metadata}, state) do
    # Add to batch for efficient processing
    turn_data = %{
      call_id: call_id,
      turn_number: turn_number,
      text: text,
      metadata: metadata,
      timestamp: System.monotonic_time(:millisecond)
    }

    new_buffer = [turn_data | state.batch_buffer]

    # Process batch if it's full or enough time has passed
    if length(new_buffer) >= state.batch_size or 
       (System.monotonic_time(:millisecond) - state.last_batch_time) > 10_000 do
      send(self(), :process_batch)
      {:noreply, %{state | batch_buffer: []}}
    else
      {:noreply, %{state | batch_buffer: new_buffer}}
    end
  end

  def handle_info(:process_batch, state) do
    if length(state.batch_buffer) > 0 do
      process_sentiment_batch(state.batch_buffer, state.model)
      
      # Record processing metrics
      :telemetry.execute(
        [:aybiza, :analytics, :sentiment, :batch_processed],
        %{count: length(state.batch_buffer)},
        %{model_version: "v2"}
      )
    end

    {:noreply, %{state | 
      batch_buffer: [], 
      last_batch_time: System.monotonic_time(:millisecond)
    }}
  end

  defp perform_sentiment_analysis(text, model) do
    # Preprocessing
    cleaned_text = preprocess_text(text)
    
    # Feature extraction
    features = extract_features(cleaned_text)
    
    # Model inference
    prediction = MLPipeline.predict(model, features)
    
    # Post-processing
    %{
      sentiment: classify_sentiment(prediction.sentiment_score),
      confidence: prediction.confidence,
      sentiment_score: prediction.sentiment_score, # -1.0 to 1.0
      emotion: predict_emotion(prediction.emotion_vector),
      topics: extract_topics(cleaned_text),
      entities: extract_entities(cleaned_text)
    }
  end

  defp process_sentiment_batch(batch, model) do
    # Batch processing for efficiency
    texts = Enum.map(batch, & &1.text)
    preprocessed_texts = Enum.map(texts, &preprocess_text/1)
    
    # Batch feature extraction
    features_batch = extract_features_batch(preprocessed_texts)
    
    # Batch prediction
    predictions = MLPipeline.predict_batch(model, features_batch)
    
    # Store results
    results = Enum.zip([batch, predictions])
    |> Enum.map(fn {turn_data, prediction} ->
      sentiment_result = %{
        sentiment: classify_sentiment(prediction.sentiment_score),
        confidence: prediction.confidence,
        sentiment_score: prediction.sentiment_score,
        emotion: predict_emotion(prediction.emotion_vector),
        topics: extract_topics(turn_data.text),
        entities: extract_entities(turn_data.text)
      }

      # Store in database
      store_conversation_analysis(turn_data, sentiment_result)
      
      sentiment_result
    end)

    Logger.info("Processed sentiment batch: #{length(results)} turns")
    results
  end

  defp preprocess_text(text) do
    text
    |> String.downcase()
    |> String.replace(~r/[^\w\s]/, " ")
    |> String.replace(~r/\s+/, " ")
    |> String.trim()
  end

  defp extract_features(text) do
    # TF-IDF features + custom voice-specific features
    %{
      tfidf_vector: compute_tfidf(text),
      length: String.length(text),
      word_count: length(String.split(text)),
      exclamation_count: String.graphemes(text) |> Enum.count(& &1 == "!"),
      question_count: String.graphemes(text) |> Enum.count(& &1 == "?"),
      caps_ratio: calculate_caps_ratio(text),
      politeness_score: calculate_politeness_score(text)
    }
  end

  defp extract_features_batch(texts) do
    Enum.map(texts, &extract_features/1)
  end

  defp classify_sentiment(score) when score > 0.1, do: "positive"
  defp classify_sentiment(score) when score < -0.1, do: "negative"
  defp classify_sentiment(_score), do: "neutral"

  defp predict_emotion(emotion_vector) do
    # Map emotion vector to emotion labels
    emotion_labels = ["happy", "sad", "angry", "frustrated", "satisfied", "confused", "excited"]
    
    emotion_vector
    |> Enum.with_index()
    |> Enum.max_by(fn {score, _idx} -> score end)
    |> elem(1)
    |> then(& Enum.at(emotion_labels, &1, "neutral"))
  end

  defp extract_topics(text) do
    # Simple keyword-based topic extraction
    # In production, use more sophisticated topic modeling
    topics = %{
      "billing" => ~r/bill|charge|payment|cost|invoice/i,
      "technical_support" => ~r/error|bug|not working|broken|fix/i,
      "account" => ~r/account|login|password|access/i,
      "product_inquiry" => ~r/product|feature|how to|information/i,
      "complaint" => ~r/complain|dissatisfied|problem|issue/i,
      "compliment" => ~r/great|excellent|amazing|wonderful|love/i
    }

    topics
    |> Enum.filter(fn {_topic, regex} -> Regex.match?(regex, text) end)
    |> Enum.map(fn {topic, _regex} -> topic end)
  end

  defp extract_entities(text) do
    # Named Entity Recognition
    # In production, integrate with spaCy or similar NER models
    %{
      phone_numbers: extract_phone_numbers(text),
      emails: extract_emails(text),
      amounts: extract_currency_amounts(text),
      dates: extract_dates(text)
    }
  end

  defp store_conversation_analysis(turn_data, sentiment_result) do
    # Store in conversation_turns table
    query = """
    UPDATE conversation_turns 
    SET 
      sentiment_score = $1,
      emotion = $2,
      entities = $3,
      updated_at = NOW()
    WHERE call_id = $4 AND turn_number = $5
    """

    entities_json = Jason.encode!(sentiment_result.entities)

    Aybiza.Repo.query(query, [
      sentiment_result.sentiment_score,
      sentiment_result.emotion,
      entities_json,
      turn_data.call_id,
      turn_data.turn_number
    ])
  end

  # Utility functions
  defp compute_tfidf(text) do
    # Simplified TF-IDF computation
    # In production, use pre-trained TF-IDF vectorizer
    words = String.split(text)
    word_frequencies = Enum.frequencies(words)
    
    # Convert to feature vector (simplified)
    word_frequencies
    |> Map.take(get_vocabulary())
    |> Map.values()
    |> Enum.map(& &1 / length(words)) # Simple term frequency
  end

  defp calculate_caps_ratio(text) do
    if String.length(text) == 0 do
      0.0
    else
      caps_count = text 
      |> String.graphemes() 
      |> Enum.count(&String.match?(&1, ~r/[A-Z]/))
      
      caps_count / String.length(text)
    end
  end

  defp calculate_politeness_score(text) do
    polite_words = ["please", "thank", "sorry", "excuse", "appreciate"]
    
    word_count = text
    |> String.downcase()
    |> String.split()
    |> Enum.count(fn word -> word in polite_words end)
    
    min(word_count / 10.0, 1.0) # Normalize to 0-1
  end

  defp get_vocabulary() do
    # In production, load from pre-trained vocabulary
    ["good", "bad", "help", "problem", "issue", "great", "terrible", "love", "hate"]
  end

  defp extract_phone_numbers(text) do
    Regex.scan(~r/\b\d{3}[-.]?\d{3}[-.]?\d{4}\b/, text)
    |> Enum.map(&hd/1)
  end

  defp extract_emails(text) do
    Regex.scan(~r/\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/, text)
    |> Enum.map(&hd/1)
  end

  defp extract_currency_amounts(text) do
    Regex.scan(~r/\$\d+(?:\.\d{2})?|\d+(?:\.\d{2})?\s*dollars?/i, text)
    |> Enum.map(&hd/1)
  end

  defp extract_dates(text) do
    Regex.scan(~r/\b\d{1,2}[\/\-]\d{1,2}[\/\-]\d{2,4}\b/, text)
    |> Enum.map(&hd/1)
  end
end
```

### 2. Predictive Analytics Engine
```elixir
defmodule Aybiza.Analytics.PredictiveEngine do
  use GenServer
  require Logger

  alias Aybiza.Analytics.{MLPipeline, TimeSeriesAnalyzer}

  def start_link(opts) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def init(_opts) do
    # Load predictive models
    {:ok, call_volume_model} = MLPipeline.load_model("call_volume_predictor_v3")
    {:ok, satisfaction_model} = MLPipeline.load_model("satisfaction_predictor_v2")
    {:ok, churn_model} = MLPipeline.load_model("customer_churn_predictor_v1")

    state = %{
      models: %{
        call_volume: call_volume_model,
        satisfaction: satisfaction_model,
        churn: churn_model
      },
      cache: %{},
      last_training: nil
    }

    # Schedule periodic model updates
    :timer.send_interval(3600_000, :update_models) # Every hour
    
    # Schedule daily predictions
    :timer.send_interval(86_400_000, :generate_daily_predictions) # Every day

    {:ok, state}
  end

  def predict_call_volume(agent_id, time_range \\ {DateTime.utc_now(), DateTime.add(DateTime.utc_now(), 86400)}) do
    GenServer.call(__MODULE__, {:predict_call_volume, agent_id, time_range})
  end

  def predict_customer_satisfaction(call_features) do
    GenServer.call(__MODULE__, {:predict_satisfaction, call_features})
  end

  def predict_customer_churn(customer_id, call_history) do
    GenServer.call(__MODULE__, {:predict_churn, customer_id, call_history})
  end

  def handle_call({:predict_call_volume, agent_id, {start_time, end_time}}, _from, state) do
    cache_key = "call_volume_#{agent_id}_#{DateTime.to_unix(start_time)}_#{DateTime.to_unix(end_time)}"
    
    prediction = case Map.get(state.cache, cache_key) do
      nil ->
        # Generate new prediction
        features = extract_call_volume_features(agent_id, start_time, end_time)
        result = MLPipeline.predict(state.models.call_volume, features)
        
        # Cache result for 30 minutes
        new_cache = Map.put(state.cache, cache_key, {result, System.monotonic_time(:second) + 1800})
        send(self(), {:update_cache, new_cache})
        
        result

      {cached_result, expiry} ->
        if System.monotonic_time(:second) > expiry do
          # Cache expired, generate new prediction
          features = extract_call_volume_features(agent_id, start_time, end_time)
          result = MLPipeline.predict(state.models.call_volume, features)
          
          new_cache = Map.put(state.cache, cache_key, {result, System.monotonic_time(:second) + 1800})
          send(self(), {:update_cache, new_cache})
          
          result
        else
          cached_result
        end
    end

    {:reply, prediction, state}
  end

  def handle_call({:predict_satisfaction, call_features}, _from, state) do
    prediction = MLPipeline.predict(state.models.satisfaction, call_features)
    {:reply, prediction, state}
  end

  def handle_call({:predict_churn, customer_id, call_history}, _from, state) do
    features = extract_churn_features(customer_id, call_history)
    prediction = MLPipeline.predict(state.models.churn, features)
    {:reply, prediction, state}
  end

  def handle_info({:update_cache, new_cache}, state) do
    {:noreply, %{state | cache: new_cache}}
  end

  def handle_info(:update_models, state) do
    # Retrain models with latest data
    Logger.info("Starting model retraining...")
    
    # Run training asynchronously
    Task.start(fn -> retrain_models() end)
    
    {:noreply, %{state | last_training: DateTime.utc_now()}}
  end

  def handle_info(:generate_daily_predictions, state) do
    # Generate predictions for all active agents
    Logger.info("Generating daily predictions...")
    
    Task.start(fn -> generate_agent_predictions() end)
    
    {:noreply, state}
  end

  defp extract_call_volume_features(agent_id, start_time, end_time) do
    # Historical call patterns
    historical_data = get_historical_call_data(agent_id, DateTime.add(start_time, -30, :day), start_time)
    
    # Time-based features
    time_features = extract_time_features(start_time, end_time)
    
    # Agent-specific features
    agent_features = get_agent_features(agent_id)
    
    # Combine all features
    %{
      # Historical patterns
      avg_daily_calls: calculate_avg_daily_calls(historical_data),
      calls_trend: calculate_trend(historical_data),
      seasonality_factor: calculate_seasonality(historical_data, start_time),
      
      # Time features
      day_of_week: time_features.day_of_week,
      hour_of_day: time_features.hour_of_day,
      is_weekend: time_features.is_weekend,
      is_holiday: time_features.is_holiday,
      
      # Agent features
      agent_performance_score: agent_features.performance_score,
      agent_capacity: agent_features.capacity,
      agent_availability: agent_features.availability,
      
      # External factors
      weather_impact: get_weather_impact(start_time),
      marketing_campaigns: get_active_campaigns(start_time)
    }
  end

  defp extract_churn_features(customer_id, call_history) do
    # Customer interaction patterns
    %{
      total_calls: length(call_history),
      avg_call_duration: calculate_avg_duration(call_history),
      satisfaction_trend: calculate_satisfaction_trend(call_history),
      resolution_rate: calculate_resolution_rate(call_history),
      escalation_rate: calculate_escalation_rate(call_history),
      time_since_last_call: calculate_time_since_last_call(call_history),
      complaint_count: count_complaints(call_history),
      sentiment_score: calculate_avg_sentiment(call_history),
      repeat_issue_count: count_repeat_issues(call_history)
    }
  end

  defp retrain_models() do
    try do
      # Fetch latest training data
      training_data = fetch_training_data()
      
      # Retrain call volume model
      Logger.info("Retraining call volume model...")
      MLPipeline.retrain_model("call_volume_predictor_v3", training_data.call_volume)
      
      # Retrain satisfaction model
      Logger.info("Retraining satisfaction model...")
      MLPipeline.retrain_model("satisfaction_predictor_v2", training_data.satisfaction)
      
      # Retrain churn model
      Logger.info("Retraining churn model...")
      MLPipeline.retrain_model("customer_churn_predictor_v1", training_data.churn)
      
      Logger.info("Model retraining completed successfully")
      
    rescue
      error ->
        Logger.error("Model retraining failed: #{inspect(error)}")
        # Send alert to monitoring system
        send_alert(:model_training_failed, error)
    end
  end

  defp generate_agent_predictions() do
    # Get all active agents
    agents = get_active_agents()
    
    tomorrow = DateTime.add(DateTime.utc_now(), 1, :day)
    
    Enum.each(agents, fn agent ->
      try do
        # Predict call volume for next 24 hours
        prediction = predict_call_volume(agent.id, {DateTime.utc_now(), tomorrow})
        
        # Store prediction
        store_prediction(agent.id, "call_volume", prediction, tomorrow)
        
        # Generate capacity recommendations
        if prediction.predicted_calls > agent.capacity * 0.8 do
          send_capacity_alert(agent.id, prediction.predicted_calls, agent.capacity)
        end
        
      rescue
        error ->
          Logger.error("Failed to generate prediction for agent #{agent.id}: #{inspect(error)}")
      end
    end)
  end

  # Utility functions for feature extraction
  defp get_historical_call_data(agent_id, start_date, end_date) do
    query = """
    SELECT 
      date_trunc('day', time) as day,
      COUNT(*) as call_count,
      AVG(customer_satisfaction_score) as avg_satisfaction
    FROM call_analytics 
    WHERE agent_id = $1 
      AND time >= $2 
      AND time < $3
    GROUP BY date_trunc('day', time)
    ORDER BY day
    """

    {:ok, result} = Aybiza.Repo.query(query, [agent_id, start_date, end_date])
    
    result.rows
    |> Enum.map(fn [day, call_count, avg_satisfaction] ->
      %{
        date: day,
        call_count: call_count,
        avg_satisfaction: avg_satisfaction || 0.0
      }
    end)
  end

  defp extract_time_features(start_time, _end_time) do
    %{
      day_of_week: Date.day_of_week(DateTime.to_date(start_time)),
      hour_of_day: start_time.hour,
      is_weekend: Date.day_of_week(DateTime.to_date(start_time)) in [6, 7],
      is_holiday: is_holiday?(DateTime.to_date(start_time)),
      month: start_time.month,
      quarter: div(start_time.month - 1, 3) + 1
    }
  end

  defp get_agent_features(agent_id) do
    query = """
    SELECT 
      AVG(customer_satisfaction_score) as performance_score,
      COUNT(*) as total_calls,
      AVG(total_duration_ms) as avg_duration
    FROM call_analytics 
    WHERE agent_id = $1 
      AND time >= NOW() - INTERVAL '30 days'
    """

    {:ok, result} = Aybiza.Repo.query(query, [agent_id])
    
    case result.rows do
      [[performance_score, total_calls, avg_duration]] ->
        %{
          performance_score: performance_score || 0.0,
          capacity: calculate_agent_capacity(total_calls, avg_duration),
          availability: 1.0 # Simplified - get from agent schedule
        }
      _ ->
        %{performance_score: 0.0, capacity: 100, availability: 1.0}
    end
  end

  defp calculate_avg_daily_calls(historical_data) do
    if length(historical_data) > 0 do
      total_calls = Enum.sum(Enum.map(historical_data, & &1.call_count))
      total_calls / length(historical_data)
    else
      0.0
    end
  end

  defp calculate_trend(historical_data) when length(historical_data) < 2, do: 0.0
  defp calculate_trend(historical_data) do
    # Simple linear trend calculation
    data_points = Enum.with_index(historical_data)
    |> Enum.map(fn {%{call_count: count}, index} -> {index, count} end)
    
    TimeSeriesAnalyzer.calculate_linear_trend(data_points)
  end

  defp calculate_seasonality(historical_data, target_time) do
    # Weekly seasonality
    target_day_of_week = Date.day_of_week(DateTime.to_date(target_time))
    
    same_day_data = Enum.filter(historical_data, fn %{date: date} ->
      Date.day_of_week(date) == target_day_of_week
    end)
    
    if length(same_day_data) > 0 do
      avg_same_day = Enum.sum(Enum.map(same_day_data, & &1.call_count)) / length(same_day_data)
      overall_avg = calculate_avg_daily_calls(historical_data)
      
      if overall_avg > 0, do: avg_same_day / overall_avg, else: 1.0
    else
      1.0
    end
  end

  defp is_holiday?(date) do
    # Simplified holiday detection
    holidays = [
      {1, 1},   # New Year's Day
      {7, 4},   # Independence Day
      {12, 25}  # Christmas
    ]
    
    {date.month, date.day} in holidays
  end

  defp calculate_agent_capacity(total_calls, avg_duration) do
    # Simplified capacity calculation based on historical performance
    if avg_duration && avg_duration > 0 do
      # Assume 8-hour work day, 80% efficiency
      daily_capacity_minutes = 8 * 60 * 0.8
      avg_call_minutes = avg_duration / (1000 * 60)
      
      round(daily_capacity_minutes / avg_call_minutes)
    else
      100 # Default capacity
    end
  end

  # Additional helper functions would be implemented here...
  defp get_weather_impact(_start_time), do: 0.0
  defp get_active_campaigns(_start_time), do: 0
  defp fetch_training_data(), do: %{call_volume: [], satisfaction: [], churn: []}
  defp get_active_agents(), do: []
  defp store_prediction(_agent_id, _type, _prediction, _date), do: :ok
  defp send_capacity_alert(_agent_id, _predicted, _capacity), do: :ok
  defp send_alert(_type, _error), do: :ok

  # Churn-related calculations
  defp calculate_avg_duration(call_history) do
    if length(call_history) > 0 do
      total_duration = Enum.sum(Enum.map(call_history, & &1.duration))
      total_duration / length(call_history)
    else
      0.0
    end
  end

  defp calculate_satisfaction_trend(call_history) do
    if length(call_history) >= 2 do
      satisfactions = Enum.map(call_history, & &1.satisfaction_score)
      |> Enum.with_index()
      
      TimeSeriesAnalyzer.calculate_linear_trend(satisfactions)
    else
      0.0
    end
  end

  defp calculate_resolution_rate(call_history) do
    if length(call_history) > 0 do
      resolved_count = Enum.count(call_history, & &1.resolution_status == "resolved")
      resolved_count / length(call_history)
    else
      0.0
    end
  end

  defp calculate_escalation_rate(call_history) do
    if length(call_history) > 0 do
      escalated_count = Enum.count(call_history, & &1.escalated == true)
      escalated_count / length(call_history)
    else
      0.0
    end
  end

  defp calculate_time_since_last_call(call_history) do
    if length(call_history) > 0 do
      last_call = Enum.max_by(call_history, & &1.timestamp)
      DateTime.diff(DateTime.utc_now(), last_call.timestamp, :day)
    else
      999 # Very high number if no calls
    end
  end

  defp count_complaints(call_history) do
    Enum.count(call_history, & &1.sentiment == "negative" and &1.intent == "complaint")
  end

  defp calculate_avg_sentiment(call_history) do
    if length(call_history) > 0 do
      total_sentiment = Enum.sum(Enum.map(call_history, & &1.sentiment_score))
      total_sentiment / length(call_history)
    else
      0.0
    end
  end

  defp count_repeat_issues(call_history) do
    # Count calls with similar intents/topics
    intent_counts = Enum.frequencies_by(call_history, & &1.intent)
    
    intent_counts
    |> Map.values()
    |> Enum.filter(& &1 > 1)
    |> Enum.sum()
  end
end
```

### 3. Business Intelligence Queries
```sql
-- Advanced analytics queries for business intelligence

-- 1. Agent Performance Analysis with ML Insights
CREATE OR REPLACE VIEW agent_performance_analysis AS
WITH agent_stats AS (
  SELECT 
    agent_id,
    DATE_TRUNC('week', time) AS week,
    COUNT(*) AS total_calls,
    AVG(customer_satisfaction_score) AS avg_satisfaction,
    AVG(total_duration_ms) / 1000.0 AS avg_duration_seconds,
    COUNT(*) FILTER (WHERE resolution_status = 'resolved') AS resolved_calls,
    AVG(voice_processing_latency_ms) AS avg_latency,
    SUM(cost_usd) AS total_revenue
  FROM call_analytics
  WHERE time >= NOW() - INTERVAL '12 weeks'
  GROUP BY agent_id, DATE_TRUNC('week', time)
),
agent_trends AS (
  SELECT 
    agent_id,
    REGR_SLOPE(avg_satisfaction, EXTRACT(EPOCH FROM week)) AS satisfaction_trend,
    REGR_SLOPE(total_calls, EXTRACT(EPOCH FROM week)) AS volume_trend,
    REGR_SLOPE(total_revenue, EXTRACT(EPOCH FROM week)) AS revenue_trend
  FROM agent_stats
  GROUP BY agent_id
)
SELECT 
  a.agent_id,
  AVG(a.avg_satisfaction) AS overall_satisfaction,
  AVG(a.total_calls) AS avg_weekly_calls,
  AVG(a.avg_duration_seconds) AS avg_call_duration,
  AVG(a.resolved_calls::FLOAT / a.total_calls) AS resolution_rate,
  AVG(a.avg_latency) AS avg_processing_latency,
  SUM(a.total_revenue) AS total_revenue,
  t.satisfaction_trend,
  t.volume_trend,
  t.revenue_trend,
  CASE 
    WHEN t.satisfaction_trend > 0.01 THEN 'improving'
    WHEN t.satisfaction_trend < -0.01 THEN 'declining'
    ELSE 'stable'
  END AS performance_trend
FROM agent_stats a
JOIN agent_trends t ON a.agent_id = t.agent_id
GROUP BY a.agent_id, t.satisfaction_trend, t.volume_trend, t.revenue_trend;

-- 2. Real-time Call Quality Monitoring
CREATE OR REPLACE VIEW real_time_quality_metrics AS
SELECT 
  time_bucket('5 minutes', time) AS time_window,
  agent_id,
  COUNT(*) AS calls_in_window,
  AVG(audio_quality_score) AS avg_audio_quality,
  AVG(voice_processing_latency_ms) AS avg_latency,
  AVG(customer_satisfaction_score) AS avg_satisfaction,
  COUNT(*) FILTER (WHERE error_count > 0) AS calls_with_errors,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY voice_processing_latency_ms) AS p95_latency
FROM call_analytics
WHERE time >= NOW() - INTERVAL '2 hours'
GROUP BY time_bucket('5 minutes', time), agent_id
ORDER BY time_window DESC;

-- 3. Customer Journey Analysis
CREATE OR REPLACE VIEW customer_journey_analysis AS
WITH customer_calls AS (
  SELECT 
    caller_number,
    time,
    call_id,
    agent_id,
    customer_satisfaction_score,
    resolution_status,
    cost_usd,
    LAG(time) OVER (PARTITION BY caller_number ORDER BY time) AS prev_call_time,
    ROW_NUMBER() OVER (PARTITION BY caller_number ORDER BY time) AS call_sequence
  FROM call_analytics
  WHERE time >= NOW() - INTERVAL '90 days'
    AND caller_number IS NOT NULL
),
customer_metrics AS (
  SELECT 
    caller_number,
    COUNT(*) AS total_calls,
    AVG(customer_satisfaction_score) AS avg_satisfaction,
    COUNT(*) FILTER (WHERE resolution_status = 'resolved') AS resolved_calls,
    SUM(cost_usd) AS total_value,
    MAX(time) AS last_call_time,
    MIN(time) AS first_call_time,
    AVG(EXTRACT(EPOCH FROM (time - prev_call_time)) / 86400.0) AS avg_days_between_calls
  FROM customer_calls
  GROUP BY caller_number
)
SELECT 
  caller_number,
  total_calls,
  avg_satisfaction,
  resolved_calls::FLOAT / total_calls AS resolution_rate,
  total_value,
  EXTRACT(DAYS FROM (NOW() - last_call_time)) AS days_since_last_call,
  EXTRACT(DAYS FROM (last_call_time - first_call_time)) AS customer_lifetime_days,
  avg_days_between_calls,
  CASE 
    WHEN avg_satisfaction < 2.0 THEN 'high_risk'
    WHEN avg_satisfaction < 3.5 AND total_calls > 3 THEN 'medium_risk'
    WHEN EXTRACT(DAYS FROM (NOW() - last_call_time)) > 60 THEN 'dormant'
    ELSE 'healthy'
  END AS customer_status
FROM customer_metrics
ORDER BY total_value DESC;

-- 4. Operational Efficiency Analysis
CREATE OR REPLACE VIEW operational_efficiency_metrics AS
WITH hourly_metrics AS (
  SELECT 
    time_bucket('1 hour', time) AS hour,
    region,
    COUNT(*) AS total_calls,
    AVG(voice_processing_latency_ms) AS avg_latency,
    COUNT(*) FILTER (WHERE error_count > 0) AS error_calls,
    SUM(cost_usd) AS total_cost,
    AVG(audio_quality_score) AS avg_audio_quality
  FROM call_analytics ca
  JOIN system_performance_metrics spm ON 
    time_bucket('1 hour', ca.time) = time_bucket('1 hour', spm.time)
    AND ca.region = spm.region
  WHERE ca.time >= NOW() - INTERVAL '7 days'
  GROUP BY time_bucket('1 hour', ca.time), region
)
SELECT 
  hour,
  region,
  total_calls,
  avg_latency,
  error_calls::FLOAT / total_calls AS error_rate,
  total_cost / total_calls AS cost_per_call,
  avg_audio_quality,
  CASE 
    WHEN avg_latency > 150 THEN 'poor'
    WHEN avg_latency > 100 THEN 'fair'
    WHEN avg_latency > 75 THEN 'good'
    ELSE 'excellent'
  END AS latency_grade,
  LAG(total_calls) OVER (PARTITION BY region ORDER BY hour) AS prev_hour_calls,
  (total_calls - LAG(total_calls) OVER (PARTITION BY region ORDER BY hour))::FLOAT 
    / LAG(total_calls) OVER (PARTITION BY region ORDER BY hour) AS call_volume_change
FROM hourly_metrics
ORDER BY hour DESC, region;

-- 5. Predictive Maintenance Alerts
CREATE OR REPLACE VIEW system_health_alerts AS
WITH recent_metrics AS (
  SELECT 
    time,
    region,
    error_rate,
    availability_pct,
    cpu_utilization_pct,
    memory_utilization_pct,
    LAG(error_rate) OVER (PARTITION BY region ORDER BY time) AS prev_error_rate,
    LAG(availability_pct) OVER (PARTITION BY region ORDER BY time) AS prev_availability
  FROM system_performance_metrics
  WHERE time >= NOW() - INTERVAL '24 hours'
)
SELECT 
  time,
  region,
  error_rate,
  availability_pct,
  cpu_utilization_pct,
  memory_utilization_pct,
  CASE 
    WHEN error_rate > 0.05 THEN 'high_error_rate'
    WHEN availability_pct < 99.0 THEN 'low_availability'
    WHEN cpu_utilization_pct > 80 THEN 'high_cpu'
    WHEN memory_utilization_pct > 85 THEN 'high_memory'
    WHEN error_rate > prev_error_rate * 2 THEN 'error_rate_spike'
    WHEN availability_pct < prev_availability - 1 THEN 'availability_drop'
    ELSE 'normal'
  END AS alert_type,
  CASE 
    WHEN error_rate > 0.1 OR availability_pct < 95.0 THEN 'critical'
    WHEN error_rate > 0.05 OR availability_pct < 99.0 OR cpu_utilization_pct > 80 THEN 'warning'
    ELSE 'info'
  END AS severity
FROM recent_metrics
WHERE error_rate > 0.02 
   OR availability_pct < 99.5 
   OR cpu_utilization_pct > 70 
   OR memory_utilization_pct > 75
ORDER BY time DESC;
```

## Dashboard Implementation

### 1. Real-time Analytics Dashboard
```elixir
defmodule AybizaWeb.AnalyticsDashboardLive do
  use AybizaWeb, :live_view

  alias Aybiza.Analytics.{MetricsCollector, PredictiveEngine}

  def mount(_params, _session, socket) do
    if connected?(socket) do
      # Subscribe to real-time metrics
      Phoenix.PubSub.subscribe(Aybiza.PubSub, "analytics:metrics")
      
      # Schedule periodic updates
      :timer.send_interval(5000, self(), :update_metrics)
    end

    {:ok, assign_initial_metrics(socket)}
  end

  def handle_info(:update_metrics, socket) do
    {:noreply, update_dashboard_metrics(socket)}
  end

  def handle_info({:new_metric, metric}, socket) do
    {:noreply, handle_new_metric(socket, metric)}
  end

  defp assign_initial_metrics(socket) do
    socket
    |> assign(:call_volume, get_call_volume_metrics())
    |> assign(:satisfaction_metrics, get_satisfaction_metrics())
    |> assign(:performance_metrics, get_performance_metrics())
    |> assign(:alerts, get_active_alerts())
    |> assign(:predictions, get_latest_predictions())
  end

  defp update_dashboard_metrics(socket) do
    socket
    |> assign(:call_volume, get_call_volume_metrics())
    |> assign(:performance_metrics, get_performance_metrics())
    |> assign(:alerts, get_active_alerts())
  end

  defp get_call_volume_metrics() do
    MetricsCollector.get_real_time_call_volume()
  end

  defp get_satisfaction_metrics() do
    MetricsCollector.get_satisfaction_trends()
  end

  defp get_performance_metrics() do
    MetricsCollector.get_system_performance()
  end

  defp get_active_alerts() do
    MetricsCollector.get_active_alerts()
  end

  defp get_latest_predictions() do
    PredictiveEngine.get_latest_predictions()
  end

  defp handle_new_metric(socket, metric) do
    case metric.type do
      "call_completed" ->
        update_call_metrics(socket, metric)
      "system_health" ->
        update_system_metrics(socket, metric)
      _ ->
        socket
    end
  end

  defp update_call_metrics(socket, metric) do
    # Update call volume and satisfaction in real-time
    current_volume = socket.assigns.call_volume
    updated_volume = Map.update!(current_volume, :current_calls, &(&1 + 1))
    
    assign(socket, :call_volume, updated_volume)
  end

  defp update_system_metrics(socket, metric) do
    # Update system performance metrics
    current_metrics = socket.assigns.performance_metrics
    updated_metrics = merge_system_metric(current_metrics, metric)
    
    assign(socket, :performance_metrics, updated_metrics)
  end

  defp merge_system_metric(current_metrics, new_metric) do
    # Logic to merge new system metric with current state
    current_metrics
  end
end
```

This comprehensive implementation provides advanced analytics and machine learning capabilities for the AYBIZA platform, enabling real-time insights, predictive analytics, and business intelligence for voice agent operations.