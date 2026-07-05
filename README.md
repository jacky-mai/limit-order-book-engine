# Limit Order Book Reconstruction & Optimal Execution Engine

## Overview

This project is a high-frequency trading simulation framework focused on reconstructing a limit order book (LOB) from tick-level data and implementing key market microstructure and optimal execution models.

It replicates real-world exchange mechanics using a price-time-priority matching system and evaluates execution strategies under different liquidity and volatility conditions.

---

## Core Components

### 1. Limit Order Book Reconstruction
- Built a price-time-priority matching engine from scratch
- Processes 10,000+ tick-level order events
- Simulates limit orders, market orders, and cancellations
- Reconstructs full market depth over time

### 2. Market Microstructure Modelling
- Implements Kyle’s Lambda to measure price impact and liquidity
- Develops two-scale realized volatility estimators
- Adjusts for microstructure noise in high-frequency data

### 3. Optimal Execution & Market Making
- Implements Almgren–Chriss optimal execution framework
- Builds Avellaneda–Stoikov market-making model
- Simulates execution strategies under varying volatility and liquidity regimes

### 4. Simulation & Validation
- Monte Carlo simulation for strategy evaluation
- Benchmarks performance against theoretical models
- Analyses trade-offs between execution cost, risk, and price impact

---

## Tech Stack

- Python
- NumPy
- Pandas
- SciPy
- Monte Carlo Simulation

---

## Key Finance Concepts

- Market microstructure
- Order book dynamics
- Price impact modelling
- Optimal trade execution
- Stochastic simulation

---

## Skills Demonstrated

- Quantitative finance modelling
- Algorithm design and system simulation
- High-frequency data processing
- Financial engineering
- Python development

---

## Future Improvements

- Integration with real exchange data feeds
- GPU acceleration for large-scale simulations
- Real-time execution simulation engine
- Strategy optimisation using reinforcement learning
