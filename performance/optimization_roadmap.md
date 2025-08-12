# CrackTrader Performance Optimization Roadmap

## Current Performance Baseline (from bench.py profiling)

### Mock Mode Results
- **Store creation**: 4,634ms (90MB) - Test fixture import overhead
- **Pandas operations**: 707ms (24MB) - Benchmark simulation
- **Pure Backtrader**: 110ms - External library baseline
- **CrackTrader overhead**: 17ms - Our actual code overhead
- **Data processing**: ~5ms - Very fast

### Sandbox Mode Results
- **Average latency**: 775ms - Includes real network calls
- **Network operations**: 200-1000ms - External Binance testnet
- **Performance score**: 56/100 - Acceptable for sandbox with network

## 🎯 Optimization Priority Analysis

### Priority 1: High Impact, Production Critical
**Historical Data Pipeline (MAJOR INSIGHT!)**
- Current: No caching enabled by default, CSV reading = 10-80s for large datasets
- **Discovery**: We already have a sophisticated caching system but it's disabled!
- Target: Enable caching by default, add Parquet support
- Impact: 90%+ time reduction for backtesting workflows
- **Immediate Action**: Enable HistoricalDataCache by default

**Network Layer Optimization**
- Current: 200-1000ms per API call
- Target: Connection pooling, request batching
- Impact: Direct trading latency reduction
- Implementation: Python asyncio improvements first, then Rust HTTP client

**Order Processing Pipeline**
- Current: 17ms CrackTrader overhead
- Target: <5ms order validation and routing
- Impact: Critical for HFT aspirations
- **Rust Candidate**: Order validation, risk checks, position calculations

### Priority 2: Medium Impact, Scalability
**Large Dataset Processing (5M+ candles)**
- Current: Pandas CSV reading = 20-50s, 1GB+ memory usage
- Target: Streaming + Parquet = <1s, <100MB memory
- Impact: Enable multi-year, multi-symbol backtesting
- **Rust Candidate**: Technical indicators (100x speedup), data I/O pipeline

**WebSocket Data Ingestion**
- Current: Python asyncio WebSocket handling
- Target: Higher throughput tick processing
- Impact: More symbols, higher frequency data
- **Rust Candidate**: Tick data parsing, OHLCV candle building

### Priority 3: Low Impact, Nice-to-Have
**Test Infrastructure**
- Current: 4.6s test setup time (FakeExchange import)
- Target: Faster test runs
- Impact: Developer experience only
- Solution: Lazy imports, smaller test fixtures

## 🦀 Rust Integration Strategy

### Phase 1: Proof of Concept (Post-Documentation)
**Target**: Order validation module
```python
# Current Python
def validate_order(symbol, size, price, order_type):
    # Validation logic ~1-2ms

# Future Rust FFI
import cracktrader_fast
result = cracktrader_fast.validate_order(symbol, size, price, order_type)  # ~0.1ms
```

**Benefits**:
- Clear performance win (10x faster)
- Isolated, testable component
- Low integration risk

### Phase 2: Data Pipeline (If Needed)
**Target**: Technical indicators for backtesting
```python
# Current (if we implemented)
df['sma_20'] = df['close'].rolling(20).mean()  # 700ms for 1000 points

# Rust alternative
sma_values = cracktrader_fast.sma(prices, window=20)  # ~10ms for 1000 points
```

### Phase 3: Network Layer (Advanced)
**Target**: HTTP client with connection pooling
- Replace Python requests/aiohttp with Rust reqwest
- Custom protocol optimizations
- Only if network becomes the bottleneck

## 🔄 Integration Approach

### Option 1: PyO3 Rust Extension
```toml
[package]
name = "cracktrader-fast"
version = "0.1.0"
edition = "2021"

[dependencies]
pyo3 = "0.20"
serde = "1.0"
tokio = "1.0"
```

### Option 2: Separate Process (microservice)
- Rust binary for intensive computations
- IPC via message queues
- Better isolation, easier deployment

### Option 3: WASM (Future consideration)
- Rust -> WebAssembly for browser compatibility
- Could enable web-based backtesting

## 📊 Success Metrics

### Before Optimization (Current)
- Order validation: ~1ms Python
- Mock tests: 1.6s total time
- Sandbox tests: 775ms average (network-bound)
- Memory usage: High pandas overhead when used

### Target After Rust Optimization
- Order validation: <0.1ms (10x improvement)
- Mock tests: <500ms total time
- Technical indicators: 50x faster than pandas
- Memory usage: Reduced allocations

## 🗓️ Timeline & Dependencies

### Prerequisites (Current Priority)
1. ✅ Complete performance benchmarking framework
2. 🔄 Finish documentation
3. 🔄 Stabilize core Python functionality
4. 🔄 Comprehensive test coverage

### Rust Development (Post-Documentation)
1. **Week 1-2**: Set up Rust development environment, PyO3 hello world
2. **Week 3-4**: Implement order validation in Rust
3. **Week 5-6**: Performance testing and integration
4. **Week 7-8**: Technical indicators (if needed)

## 🤔 Key Questions to Resolve Later

1. **Do we actually need pandas-like operations?**
   - Current code doesn't use pandas heavily
   - Most technical analysis could be done in Backtrader

2. **What's our HFT ambition level?**
   - Sub-millisecond: Need Rust/Go network layer
   - Sub-10ms: Python optimization may suffice
   - Sub-100ms: Current performance probably fine

3. **Deployment complexity vs performance gain**
   - Rust extensions add build complexity
   - Cross-platform distribution challenges
   - Is the performance gain worth it?

## 💡 Immediate Actions (Low Effort, High Impact)

1. **Profile real trading scenarios** (not benchmark simulations)
2. **Optimize Python imports** (lazy loading)
3. **Connection pooling** for REST API calls
4. **Memory profiling** of actual trading sessions

## Notes
- Store creation slowness is test infrastructure, not production concern
- Pandas operations were benchmark artifacts, not real bottlenecks
- Our actual CrackTrader overhead is only 17ms - quite good!
- Network latency (200-1000ms) dominates real trading performance
