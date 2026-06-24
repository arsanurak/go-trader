# Quantitative Trading Resources

## Knowledge

- **Book: _Quantitative Trading_ — Ernest P. Chan (Wiley, 2nd ed. 2021)**
  The canonical beginner text for systematic trading. Written by a practitioner, not an academic. Covers edge, backtesting methodology, common pitfalls, and what distinguishes real strategies from coincidences. Use for: Lessons 1–5 (foundations), backtest interpretation, strategy evaluation.

- **Book: _Advances in Financial Machine Learning_ — Marcos López de Prado (Wiley, 2018)**
  Advanced. The backtesting and feature engineering chapters are essential reading — they name every way a backtest can lie to you. Use for: Understanding overfitting, the combinatorial purged cross-validation method, regime labelling. Do not start here; return after the foundations are solid.

- **Article series: "You're Backtesting Wrong" — Quant Stack Exchange / various**
  No single URL — this is a category of content. Search "backtest overfitting walk-forward" on Quant Stack Exchange for rigorous practitioner answers. Use for: Deepening any lesson on backtest validity.

- **Documentation: QuantConnect LEAN Engine**
  https://www.quantconnect.com/docs/v2/writing-algorithms
  The open-source backtesting engine behind QuantConnect. Strong pedagogical documentation with worked examples. Different framework from go-trader but the concepts are identical. Use for: seeing how other serious systems structure signals, position sizing, and execution logic.

## Wisdom (Communities)

- **r/algotrading** — https://reddit.com/r/algotrading
  High-signal subreddit. Good for: seeing what kinds of edges practitioners are actually finding, strategy critique threads, backtesting methodology debates. Filter for "discussion" flair.

- **Quant Stack Exchange** — https://quant.stackexchange.com
  Academic/practitioner Q&A. Best for: rigorous answers to mathematical questions about edge, Sharpe ratio interpretation, regime modelling. High trust — answers are upvoted by peers.

## Gaps

- No good beginner-level resource on the specific HMM/GMM regime detection approach used in this codebase. When covering regime models in depth, supplement with the sklearn HMM documentation and the relevant papers (Baum-Welch algorithm).
- No resource specifically covering ATR-based exits and sizing. The go-trader codebase itself is the best reference here.
