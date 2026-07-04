---
source_url: https://www.uber.com/nl/en/blog/scaling-real-time-traffic/
title: Scaling Real-Time Traffic Forecasting with a Graph-Aware Transformer
authors:
  - Olcay Cirit (Research Scientist)
  - Yogesh Luthra (Machine Learning Engineering Lead)
  - Niru Agrawal (Senior ML Eng Manager)
date: May 19, 2026
---

# Scaling Real-Time Traffic Forecasting with a Graph-Aware Transformer

## Introduction

When you request an Uber, the app does two things right away: it chooses a route to your destination and estimates your arrival time. Both depend on accurate, up-to-date traffic forecasts for each leg of your trip.

Uber's traffic forecasting stack has reliably supported route selection and arrival time estimation at massive scale for over a decade. To unlock further quality improvements, the Traffic Forecasting and Applied AI teams launched a multi-year effort to rebuild this system.

The result is DeepETT (Deep Estimated Travel Time), a real-time traffic forecasting system powered by deep learning that improves long-trip arrival time accuracy by 6%, boosts forecast variance explained by 19%, and drives an estimated $100 million in incremental annual revenue. Today it's also one of Uber's highest throughput deep learning deployments, serving upwards of 2 million real-time forecasts per second.

In this blog, we share the key engineering decisions behind DeepETT, the modeling choices that made it a success, and some surprising challenges we encountered along the way.

## Traffic at Uber

Uber's traffic forecasting system turns raw GPS signals from driver's phones into real-time predictions of how fast every road segment in the world is moving, and how that'll change over the next few hours.

Uber receives GPS pings roughly every 4 seconds from each trip, amounting to tens of billions of location updates per day. These pings are mapped onto a global road graph with around 100 million road segments. Each traversal of a segment produces a precise measurement of how long that stretch of road took to drive.

From this data, Uber continuously forecasts *estimated traversal times (ETTs)* for every road segment, updating them every few minutes across multiple future horizons. When a rider requests a trip, the routing engine uses the latest ETTs to find the fastest route, and the sum of the segment-level ETTs along that route forms an initial arrival time estimate.

This traffic forecasting layer sits upstream of nearly everything else: routing, pricing, arrival times, and navigation instructions given to drivers. Improvements here compound downstream. Better traffic forecasts lead to better routes, more accurate arrival times, and ultimately a better experience for both riders and drivers.

At Uber's scale, even small improvements in segment-level accuracy can translate into meaningful gains across millions of trips every day.

![Flowchart detailing Uber's route and ETA calculation process: starts with the Uber Driver App sending GPS pings for driver location updates, proceeds to Map Matching, then Traffic Forecasting using observed traversal times (streaks) and estimated traversal times (ETT forecasts) per road segment, followed by Route Finding for fastest routes and unrefined ETAs, and ends with Refinement/Calibration to produce fastest routes, refined ETAs, and fares.](https://cn-geo1.uber.com/image-proc/resize/udam/format=auto/srcb64=aHR0cHM6Ly90Yi1zdGF0aWMudWJlci5jb20vcHJvZC91ZGFtLWFzc2V0cy9mMTgwZDA3ZC1kMmFiLTQwNWUtOWIzMy1iMmM1ZjI1NWI0NTkucG5n)

*Figure 1: An overview of Uber's traffic, routing, and arrival time estimation stack*

## Design Goals for the New System

Uber's prior traffic forecasting approach was designed to be simple, stable, and scalable. It blended recent observations with historical patterns, which is computationally efficient and a very strong baseline.

As Uber expanded globally, we increasingly cared about improving traffic forecasts in three settings:

- Rapidly changing conditions, such as incidents, weather, and major events, where traffic can shift quickly.
- Sparse roads outside dense urban cores, where observations are less frequent and forecasts benefit from stronger generalization.
- Longer trips, where small differences at the segment level can compound across many segments and affect the final arrival time estimate.

Meeting these requirements called for a forecasting system that could adapt quickly, generalize across very different geographies, and learn from the full volume of available data at Uber's scale. That made traffic forecasting a strong candidate for a deep learning based rebuild.

## Scoping Decisions that De-Risked the Project

Before we started modeling, we made two scoping calls that kept DeepETT shippable. Both decisions traded theoretical elegance for something we could reliably deploy worldwide.

### Decision 1: Don't Train "End-to-End" with Routing in the Loop

Traffic forecasting feeds routing, so one option was to put route finding inside the training loop and optimize directly for trip outcomes. In theory, that's appealing.

In practice, Uber's routing stack is already mature and heavily optimized. Rebuilding (or differentiating through) routing would have dramatically expanded the project scope and risk. So we chose a simpler boundary: DeepETT predicts segment-level ETTs, routing consumes them.

![Two workflow diagrams compare decoupled and joint training for route prediction. The top workflow (Option B) shows historical and real-time traffic features feeding into a traffic-forecast model, then segment-time predictions, and finally a route-finding engine. The bottom workflow (Option A) adds a feedback loop: after the route-finding engine, predicted and actual route-level times are compared, and training loss is fed back to improve the model.](https://cn-geo1.uber.com/image-proc/resize/udam/format=auto/srcb64=aHR0cHM6Ly90Yi1zdGF0aWMudWJlci5jb20vcHJvZC91ZGFtLWFzc2V0cy82MGI5ZGExOC05NTFiLTRmYTQtOTlhZS04NzA1MTU0Yzk0Y2UucG5n)

*Figure 2: Two possible problem formulations.*

### Decision 2: Use a Fixed-Size Input Instead of a Dynamic Graph

We also had to choose how the model would consume traffic observations. Dynamic compute graphs like GNNs or sequence models require materializing a highly variable number of raw observations per example. At our scale, that typically forces aggressive downsampling at training and inference time.

![Flowchart comparing dynamic compute graphs (GNNs/Sequence Transformers) and fixed-size compute graphs (pre-aggregated observations) for DeepETT. Dynamic graphs: pros—learns from raw observations, high expressiveness; cons—must load raw data, memory/latency heavy, requires down sampling, variable batch sizes. Fixed-size graphs: pros—use all data, predictable latency/throughput, scales to production; cons—limited expressiveness by aggregation, extra offline feature engineering.](https://cn-geo1.uber.com/image-proc/resize/udam/format=auto/srcb64=aHR0cHM6Ly90Yi1zdGF0aWMudWJlci5jb20vcHJvZC91ZGFtLWFzc2V0cy8wYTJkOWEwYS01OGJjLTQzZjEtYmU5Ni0wOGY3NjRhOTg5OGYucG5n)

*Figure 3: Considerations in fixed versus dynamic compute graphs.*

## Defining the Contracts

Traffic forecasts sit upstream of routing, ETAs, pricing, and driver navigation, so we had to be explicit about what we were promising before we wrote model code. We framed DeepETT as a system with two contracts:

- Segment-level forecasts. Every 5 minutes, forecast the mean estimated traversal time (ETT) for each road segment, out to a three-hour horizon. We measure accuracy at the segment level using mean squared error (MSE).
- Trip-level forecasts. When the routing engine sums segment ETTs along the fastest route, the resulting unrefined trip arrival time should have lower error than the legacy system. We measure accuracy at the trip level using mean absolute error (MAE).

The first contract is the one the model directly serves: callers provide a segment ID, timestamp, and horizon, and get back an ETT in milliseconds. Optimizing MSE here is deliberate since routing needs an expected segment traversal time to make consistent fastest path decisions.

The second contract is trickier because we don't train inside the routing loop. We can evaluate trip-level arrival times offline, but we don't directly optimize them during training. And there's a deeper mismatch: even if every segment prediction improves under MSE, the sum of those predictions isn't guaranteed to improve under MAE at the trip level. In fact, it's possible for segment MSE to go down while trip MAE gets worse, which is an important paradox that shaped the rest of the system design.

## The Crucial Role of Calibration and Resolution

To make sense of the segment-level vs. trip-level paradox, we found it useful to decompose our metrics into two independent components: resolution and calibration.

Resolution is information. It measures the fraction of real-world variability the forecast explains. We report this as variance explained (%).

Calibration is systematic error. It captures effects like predictions being consistently too high or too low. Calibration error can be arbitrarily large, but easily fixed by simply calibrating the model.

Figure 4 shows what this looked like for DeepETT. At the segment level, our model starts at around 30% variance explained, and as expected, resolution gradually decays as we forecast further into the future.

![A blue line graph with data points marked and labeled, showing a slight downward trend from approximately 0.3 to 0.28 as the x-axis increases from 0 to 60.](https://cn-geo1.uber.com/image-proc/resize/udam/format=auto/srcb64=aHR0cHM6Ly90Yi1zdGF0aWMudWJlci5jb20vcHJvZC91ZGFtLWFzc2V0cy84MGZjZWNmMy0zODAzLTRjZTYtOWZiYS00MGU3OGI4M2FjMTkucG5n)

*Figure 4: Segment-level variance explained by forecast interval (1.0 is best).*

When we next looked at trip-level MAE decompositions by summing forecasts along real driver routes, what we saw surprised us:

- Resolution jumped from ~30% (segment) to ~85% (trip)
- Millisecond-scale gains in segment-level resolution compounded into single-digit percentage gains in trip level resolution
- Trip-level calibration error was non-trivial, but we assumed that downstream calibration systems would correct this automatically.

These results gave us confidence to prioritize resolution during model iteration. If we could keep adding resolution at the segment level, it'd translate into better routing choices and better arrival times once calibrated.

## Model Design

### Model Contract

DeepETT is designed as a drop-in replacement for the legacy traffic forecaster: given a road segment, a timestamp, and a forecast horizon (0–180 minutes), it returns an expected traversal time for that segment. This simple contract keeps DeepETT decoupled from routing and downstream ETA models, and it makes inference predictable.

### Data Pipeline Design

DeepETT's data pipeline design hinges on a core assumption: given sufficient local traffic information on and around a segment, traversal times are conditionally independent of more distant segments. This means forecasting for a segment in San Francisco doesn't require knowing what's happening in New York or Los Angeles—local observations suffice.

Each segment has a local receptive field that aggregates observations over a fixed set of spatiotemporal views. These views deliberately cover multiple fallback levels so the model still has signal when observations are sparse:

- Spatial views: the segment itself, nearby road-graph neighborhoods, and broader geographic regions
- Temporal views: fresh real-time aggregates (e.g., last hour), historical patterns (days → weeks), and long-term baselines
- Context features: time-of-week, holidays/events, and other known drivers of periodic traffic shifts

For dense, frequently traveled roads, the model can lean heavily on recent segment-level observations. For sparse roads (common outside major cities), it can shift attention toward graph neighborhood or regional aggregates. The key idea is that the model always receives the same shape of input, but can learn which views are most useful in any given situation.

![Matrix diagram with three columns labeled 'Road Segments,' 'Graph Neighborhoods,' and 'Geographical Regions,' and three rows labeled 'Static Embeddings,' 'Historical Aggregations,' and 'Real-time Aggregations.' Each cell is shaded to indicate intersections between these categories.](https://cn-geo1.uber.com/image-proc/resize/udam/format=auto/srcb64=aHR0cHM6Ly90Yi1zdGF0aWMudWJlci5jb20vcHJvZC91ZGFtLWFzc2V0cy9lZjBhNzI5Yy0xM2VmLTQ0M2UtYmYwYS00NDIwMGViNDllYTAucG5n)

*Figure 5: A matrix showing spatial and temporal views of traffic data where each grid cell represents a distinct way of summarizing traffic observations.*

![Four quadrants labeled Historical, Geographical, Real-time, and Road Graph, each containing abstract representations: bar charts for time periods in Historical and Real-time, a grid of squares for Geographical, and a network of connected nodes for Road Graph.](https://cn-geo1.uber.com/image-proc/resize/udam/format=auto/srcb64=aHR0cHM6Ly90Yi1zdGF0aWMudWJlci5jb20vcHJvZC91ZGFtLWFzc2V0cy9hNzE2ZWY4YS0wYjViLTQ0MzUtODM5Yy04Zjg3YzAyYzEyZGIucG5n)

*Figure 6: Different aggregation granularities over space and time.*

### Model Architecture

With a fixed, multi-view input, the model's job is to combine many views of the segment into the best possible forecast and do it cheaply at inference time.

DeepETT uses a transformer architecture that treats each spatiotemporal view as a token: we quantize and embed the aggregated features from each view, concatenate them and pass them through multiple transformer-style blocks. This lets the model learn interactions like:

- Real-time slowdown on the segment matters more than the usual weekly pattern right now, or
- This segment is sparse—rely on neighborhood and regional traffic instead.

A single prediction head then produces an ETT conditioned on the requested forecast horizon. We do this by embedding the horizon and time context as model inputs, so the same model can answer "right now" and "three hours from now" queries.

![Flowchart illustrating a machine learning model for spatiotemporal feature inputs, including static road attributes, historical traversal stats, real-time segment stats, graph neighborhood, timeline, holiday/event countdowns, and forecast horizon embedding. Features are processed through quantization, hashing, and embedding lookup, concatenated, then passed through a stack of transformers and fully connected layers, ending with an ETT prediction head using mean squared error.](https://cn-geo1.uber.com/image-proc/resize/udam/format=auto/srcb64=aHR0cHM6Ly90Yi1zdGF0aWMudWJlci5jb20vcHJvZC91ZGFtLWFzc2V0cy8wNGI4NGE3Ny04ZmYwLTQxYmEtYmI2OS1mMGY1NTc5MWM2YWIucG5n)

*Figure 7: DeepETT model architecture diagram.*

Under the hood, we lean on a few scaling-friendly embedding strategies:

- High-cardinality IDs (segment IDs, S2 cells) use feature hashing and in-model embeddings
- Graph Embeddings for each segment are pre-computed in a separate message-passing step
- Graph Features are pre-computed over multi-hop neighborhoods around each segment
- Time is represented with a minute-of-week embedding in local time
- Holidays / special events are captured with event embeddings plus "days until / since" features

The overall effect is a model that gets many of the benefits people reach for with GNNs while keeping inference constant-time and production-friendly.

## Robust Production Pipelines

To implement DeepETT worldwide, we transformed our offline prototypes into production data pipelines capable of operating at Uber's extensive scale.

We use Apache Spark™ pipelines to precompute historical and weekly feature aggregates. Apache Flink® pipelines deliver real-time data updates every few minutes, enabling dynamic forecasts.

![Flowchart showing two main sections: Batch pipelines (yellow background) and Realtime Pipelines (green background). Batch pipelines combine Static, Graph, Geo, and Segment data through a batch feature joiner, producing Batch features. These are sent to an MA server, which receives a response join key and outputs a batch of predictions to RTTrafficMAPPredictionClient. Realtime Pipelines combine Graph, Geo, and Segment data through a realtime feature joiner, outputting to Hive storage for offline training. RTTrafficMAPPredictionClient sends a batch of palette join keys, basis features, and realtime features to Hive storage.](https://cn-geo1.uber.com/image-proc/resize/udam/format=auto/srcb64=aHR0cHM6Ly90Yi1zdGF0aWMudWJlci5jb20vcHJvZC91ZGFtLWFzc2V0cy8xMzJiNTE0NS0yNjY1LTQxMTktOTkzZC0wZWRlMDU4YmNkZDUucG5n)

*Figure 8: Production data pipelines for DeepETT.*

Our prediction-serving infrastructure was scaled to manage billions of daily predictions without any downtime, achieving:

- Real-time ingestion exceeding 160,000 feature rows per second.
- Global serving of approximately 2 million segment-level predictions per second, the highest throughput model at Uber.

## Real-Time Calibration in Practice

Downstream of DeepETT, trip-level arrival time models receive an unrefined arrival time from the routing engine and combine it with other information to produce a final, rider-facing arrival time. When we retrained these downstream models using our new traffic forecasts, we observed a decrease in final arrival time accuracy. This was highly unexpected, given the increased resolution of our traffic forecasts and unrefined arrival times.

The root cause was calibration drift. Immediately after each weekly DeepETT retraining, the aggregate calibration error was low, but the calibration curve (predicted versus observed travel time) kept shifting during the following week, and the direction and magnitude of that shift varied by city and time of day. Small segment‑level miscalibrations snowballed into much larger trip‑level errors, meaning that the model's forecasting contract wasn't consistently honored.

![Two line graphs compare three datasets. The left graph shows three colored lines (yellow, red, purple) with significant upward and downward trends, indicating notable changes over time. The right graph displays the same three lines, but all remain nearly flat, suggesting minimal variation.](https://cn-geo1.uber.com/image-proc/resize/udam/format=auto/srcb64=aHR0cHM6Ly90Yi1zdGF0aWMudWJlci5jb20vcHJvZC91ZGFtLWFzc2V0cy81NTc3MGUyOC0wOWMxLTRjMzMtOGVjNy00NzJhOTQzYTBmNzEucG5n)

*Figure 9: Illustrative segment-level residuals for different weeks before and after calibration.*

![Two side-by-side line graphs with five colored lines each. The left graph shows lines diverging upward and downward from the center, while the right graph shows lines closely clustered around the center, indicating less variation.](https://cn-geo1.uber.com/image-proc/resize/udam/format=auto/srcb64=aHR0cHM6Ly90Yi1zdGF0aWMudWJlci5jb20vcHJvZC91ZGFtLWFzc2V0cy9mNDJkNmMxYi01NzgyLTRkYjctOGQ4ZS1mNDRlNTY4NjBjZTYucG5n)

*Figure 10: Illustrative trip-level residuals before and after segment-level calibration.*

A perfectly calibrated model has flat residuals hugging the y=0 line. When the curve wanders week-over-week, downstream models interpret the same numeric forecast differently from one day to the next, degrading performance. If you only calibrate the model once at training time, the forecasts could easily drift out of SLA after deployment in unpredictable ways, so we needed a solution that continuously calibrates forecasts.

### Our Solution: Real‑Time Calibration

We built a Flink pipeline that:

- Buckets forecasts for each city into 10‑minute travel‑time bins.
- Joins those buckets with the most recent observed traversal times.
- Learns and applies a real-time correction that drives segment‑level calibration error toward zero.

![Flowchart showing the process of generating corrected forecasts. Segment forecasts lead to bucket forecasts (10 ms bins), which are joined with observed traversal times. The joined data is used to learn and apply real-time correction, resulting in corrected forecasts.](https://cn-geo1.uber.com/image-proc/resize/udam/format=auto/srcb64=aHR0cHM6Ly90Yi1zdGF0aWMudWJlci5jb20vcHJvZC91ZGFtLWFzc2V0cy8zNGY5NGE4YS0wZDliLTRmMjUtYTQ3OS01Zjg5OGY3ZDBjN2IucG5n)

*Figure 11: Real-time calibration data flow diagram.*

This real-time pipeline locked the calibration curve in place, keeping trip-level residuals flat and constant even through high‑traffic events such as NYE in New York City, while still preserving all of the resolution gains from DeepETT's traffic forecasts.

## Impact: $100 Million Annualized Value

In a comprehensive global A/B test, DeepETT substantially improved rider experiences worldwide, especially on critical longer trips:

- Long-trip arrival time accuracy improved by 6%
- Forecast variance explained increased by 19%
- Navigation defects decreased by 2.73%
- Overall mobility gross bookings uplift of $100 million annually

![Two side-by-side maps display routes through São Paulo, Brazil, with highlighted paths connecting Jardim América in the southwest to the Campo de Marte Airport in the north. Blue markers indicate points of interest or stops along each route, with the left map showing a slightly different path and more clustered markers near the airport compared to the right map.](https://cn-geo1.uber.com/image-proc/resize/udam/format=auto/srcb64=aHR0cHM6Ly90Yi1zdGF0aWMudWJlci5jb20vcHJvZC91ZGFtLWFzc2V0cy82YTFiNmRkMS1kMjllLTRlNWYtOWExYS1mMzdmNjFkMDlhNTAucG5n)

*Figure 12: Example routes selected using the baseline traffic forecasts (left) and DeepETT forecasts (right) during the A/B test.*

## Conclusion

To wrap up this blog, here are some of the lessons learned from developing DeepETT.

### Define Scope and Contracts Upfront

In software engineering and in forecasting, contracts are essential. Deciding up front what we did and didn't intend to do with deep learning mitigated technical risk and prevented endless scope creep. Defining two hard metrics—segment-level MSE and trip-level MAE—before a single line of code kept engineering, science, and product fully aligned.

### Focus on Resolution

The calibration-resolution decomposition helped our team estimate early on the lift our model would provide on systems far downstream. Resolution rigorously quantified the information content of our forecasts at each forecast interval. Even small segment-level resolution gains in the milliseconds compounded into multi-second trip-level resolution wins.

### Always Be Calibrating

A miscalibrated forecast is by definition not upholding its contract. Calibrating once at training time isn't enough, post-training calibration drift can degrade system performance, especially in cases where one model feeds into another. Inevitable calibration errors can be definitively solved post-training with continual, real-time calibration.

### System Design Drives Model Architecture

We chose to pre-aggregate raw observations into a comprehensive set of spatio-temporal views (segment, graph, region, hourly, weekly, event) to preserve relevant information with no downsampling while offering fast, constant-time inference at a global scale. Had we opted for a classical GNN model architecture, we may have seen good offline gains but later faced challenges productionizing.

As we look ahead, DeepETT is an ongoing journey toward perfecting traffic predictions, improving millions of trips daily.

## Acknowledgments

The authors would like to thank the following individuals who contributed immeasurably to this project:

Traffic Engineering: Sibi AR, Saandeep Depatla, and Scott Sum.

Applied AI: Reza Asadi and Himaanshu Gupta.

Applied Science: Zhounan Li and Helin Zhu.

Product: Sang Park and Achal Gupta.

AI Platform: Rush Tehrani, Haarith Devarajan, Divya Nagar, and Nicholas Marcott.

Special thanks to Tanmay Binaykiya, Michael Mallory, Chao-lin Cho and Harshit Dubey.

Cover Photo Attribution: "New York City" by Earth Science and Remote Sensing Unit, NASA Johnson Space Center.

Apache®, Apache Spark™, Flink®, and the star logo are either registered trademarks or trademarks of the Apache Software Foundation in the United States and/or other countries. No endorsement by The Apache Software Foundation is implied by the use of these marks.