# Teaching Notes

## User profile
- Software engineer, deep Go and Python familiarity
- Works with go-trader codebase daily: reads code, applies changes, observes output
- Quant theory level: beginner — can follow the code logic but hasn't studied the underlying statistical concepts
- Learns well from concrete anchors: code references, worked numbers, named components over abstract frameworks

## Preferences (discovered)
- Ground every abstract concept in the go-trader codebase — name the actual file/component
- Math notation is fine; don't simplify the formulas, just explain them well
- Worked numerical examples are valuable
- **All lessons must be written in Thai** — translate all prose, labels, quiz text, and UI into Thai; keep code identifiers, file paths, and technical terms (Edge, Sharpe, regime, backtest, etc.) in English

## Teaching approach
- Use the codebase as a running reference — each lesson should name at least one go-trader component and connect it to the lesson concept
- Build from expected value up (the whole pipeline traces back to EV)
- Introduce tools (Sharpe, regime labels, circuit breaker) by function before by mechanism
