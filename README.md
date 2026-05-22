# mock-int

# Limit Order Book Matching Engine

  **Time:** 60–90 min. Pair programming. No finance knowledge needed.

  ---
  
  ## The Problem

  Build the data structure behind a stock exchange. Traders send **limit orders**:

  - **Buy:** "I'll buy N shares for **up to** $P each."
  - **Sell:** "I'll sell N shares for **at least** $P each."

  When a buyer's max meets a seller's min, a **trade** happens. Unmatched orders rest in the **order book** until matched
  or canceled.

  ```
    ASKS (sellers, lowest first)
    $102 → 6
    $101 → 4
    $100 → 3       ← best ask
    ────────────
    $99  → 5       ← best bid
    $98  → 10
    BIDS (buyers, highest first)
  ```

  A **price level** is all orders at the same price on the same side.

  ---
  
  ## Matching Rules

  1. New **buy at P** matches resting sells with price `≤ P`, **lowest price first**.
  2. New **sell at P** matches resting buys with price `≥ P`, **highest price first**.
  3. Within a price level, fill **FIFO** (earliest order first).
  4. Trade executes at the **resting** order's price (the aggressor may get price improvement).
  5. Partial fills are allowed. Unfilled remainder rests on the book.
  6. When the last order at a price level leaves, the level disappears.

  **Price improvement example:** Book has a resting `SELL 5 @ $100`. New `BUY 5 @ $105` arrives → trade executes at
  **$100**.

  ---

  ## Walked Example

  Start with empty book.

  | # | Event | Result | Book after |
  |---|---|---|---|
  | 1 | Alice: `SELL 5 @ $100` (id=A) | rests | `ASKS: $100→5` |
  | 2 | Bob: `BUY 3 @ $99` (id=B) | rests | `ASKS: $100→5` / `BIDS: $99→3` |
  | 3 | Carol: `BUY 7 @ $101` (id=C) | `Trade(buy=C, sell=A, $100, 5)`; remaining 2 rests | `BIDS: $101→2, $99→3` |
  | 4 | cancel `B` | `True` | `BIDS: $101→2` |

  ---
  
  ## What to Implement

  ### `Side` (enum)
  `BUY`, `SELL`

  ### `Order` (value type)
  | Field | Type |
  |---|---|
  | `order_id` | `str` (unique) |
  | `side` | `Side` |
  | `price` | `int` (> 0) |
  | `quantity` | `int` (> 0) |

  Constructor raises `ValueError` if `price ≤ 0` or `quantity ≤ 0`.
  
  ### `Trade` (value type)
  | Field | Type |
  |---|---|
  | `buy_order_id`, `sell_order_id` | `str` |
  | `price`, `quantity` | `int` |

  ### `PriceLevelSnapshot`
  `price: int`, `total_quantity: int`, `order_count: int`

  ### `OrderBookSnapshot`
  `bids: List[PriceLevelSnapshot]` (descending), `asks: List[PriceLevelSnapshot]` (ascending)

  ### `OrderBook`
  | Method | Returns | Notes |
  |---|---|---|
  | `place_order(order)` | `List[Trade]` | Match against book; rest the remainder. Raises `ValueError` on duplicate
  `order_id`. |
  | `cancel_order(order_id)` | `bool` | `False` if not resting (already filled or never existed). |
  | `get_best_bid()` | `Optional[int]` | `None` if empty. |
  | `get_best_ask()` | `Optional[int]` | `None` if empty. |
  | `get_depth(levels)` | `OrderBookSnapshot` | Top N levels per side, aggregated. |

  ---
  
  ## Example Usage

  ```python
  book = OrderBook()
  book.place_order(Order("A", Side.SELL, 100, 5))   # []
  book.place_order(Order("B", Side.BUY,  99,  3))   # []
  book.place_order(Order("C", Side.BUY,  101, 7))   # [Trade(C, A, 100, 5)]

  book.get_best_bid()    # 101
  book.get_best_ask()    # None
  book.cancel_order("B") # True
  book.cancel_order("A") # False  (already filled)
  ```
  
  ---

  ## Constraints

  - Single symbol, single-threaded, in-memory.
  - Target **O(log n)** for `place_order` and `cancel_order` (n = price levels).
  - Prices/quantities are positive integers (treat as cents).
  
  **Out of scope:** market orders, multiple symbols, persistence, thread safety, advanced order types (stop, iceberg),
  self-trade prevention.

  ---

  ## What You're Evaluated On

  1. **Correctness** — right trades, right order, right prices.
  2. **Class design** — clear responsibilities, no god classes.
  3. **Data structures** — justify fast best-price lookup, FIFO-within-price, cancel-by-id.
  4. **Edge cases** — empty book, duplicate ids, canceling filled orders, multi-level sweeps, partial fills.
  5. **Communication** — clarifying questions, explaining tradeoffs as you go.
