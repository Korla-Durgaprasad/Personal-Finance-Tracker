from __future__ import annotations
from dataclasses import dataclass, asdict, field
from datetime import datetime
from typing import List, Optional, Dict
import json
import os
from itertools import groupby
from operator import itemgetter

# -----------------------------
# Data Models
# -----------------------------

@dataclass
class Transaction:
    id: int
    date: str          # ISO format: YYYY-MM-DD
    type: str          # "INCOME" or "EXPENSE"
    category: str
    amount: float      # positive value; sign is implied by type
    note: str = ""

    def to_dict(self):
        return asdict(self)

    @staticmethod
    def from_dict(d: dict) -> "Transaction":
        return Transaction(
            id=d["id"],
            date=d["date"],
            type=d["type"],
            category=d["category"],
            amount=float(d["amount"]),
            note=d.get("note", "")
        )

@dataclass
class SavingsGoal:
    id: int
    name: str
    target_amount: float
    current_amount: float
    due_date: Optional[str] = None  # ISO date or None

    def to_dict(self):
        return asdict(self)

    @staticmethod
    def from_dict(d: dict) -> "SavingsGoal":
        return SavingsGoal(
            id=d["id"],
            name=d["name"],
            target_amount=float(d["target_amount"]),
            current_amount=float(d["current_amount"]),
            due_date=d.get("due_date")
        )

@dataclass
class DataStore:
    transactions: List[Transaction] = field(default_factory=list)
    goals: List[SavingsGoal] = field(default_factory=list)

    def to_dict(self):
        return {
            "transactions": [t.to_dict() for t in self.transactions],
            "goals": [g.to_dict() for g in self.goals]
        }

    @staticmethod
    def from_dict(d: dict) -> "DataStore":
        ts = [Transaction.from_dict(td) for td in d.get("transactions", [])]
        gs = [SavingsGoal.from_dict(gd) for gd in d.get("goals", [])]
        return DataStore(transactions=ts, goals=gs)

# -----------------------------
# Persistence
# -----------------------------

DATA_FILE = "finance_data.json"

def save_store(store: DataStore, path: Optional[str] = None) -> None:
    path = path or DATA_FILE
    with open(path, "w", encoding="utf-8") as f:
        json.dump(store.to_dict(), f, indent=2)

def load_store(path: Optional[str] = None) -> DataStore:
    path = path or DATA_FILE
    if not os.path.exists(path):
        return DataStore()
    with open(path, "r", encoding="utf-8") as f:
        data = json.load(f)
    return DataStore.from_dict(data)

# -----------------------------
# Core Operations
# -----------------------------

class FinanceEngine:
    def __init__(self, store: Optional[DataStore] = None):
        self.store = store or DataStore()

    # Helpers
    def _next_id(self, items: List) -> int:
        if not items:
            return 1
        return max(item.id for item in items) + 1

    # Add
    def add_income(self, date: str, category: str, amount: float, note: str = "") -> Transaction:
        t = Transaction(
            id=self._next_id(self.store.transactions),
            date=date,
            type="INCOME",
            category=category,
            amount=float(amount),
            note=note,
        )
        self.store.transactions.append(t)
        return t

    def add_expense(self, date: str, category: str, amount: float, note: str = "") -> Transaction:
        t = Transaction(
            id=self._next_id(self.store.transactions),
            date=date,
            type="EXPENSE",
            category=category,
            amount=float(amount),
            note=note,
        )
        self.store.transactions.append(t)
        return t

    def add_goal(self, name: str, target_amount: float, current_amount: float = 0.0, due_date: Optional[str] = None) -> SavingsGoal:
        g = SavingsGoal(
            id=self._next_id(self.store.goals),
            name=name,
            target_amount=float(target_amount),
            current_amount=float(current_amount),
            due_date=due_date
        )
        self.store.goals.append(g)
        return g

    # Update / Delete
    def update_transaction(self, t_id: int, **kwargs) -> bool:
        for t in self.store.transactions:
            if t.id == t_id:
                for k, v in kwargs.items():
                    if hasattr(t, k):
                        setattr(t, k, v)
                return True
        return False

    def delete_transaction(self, t_id: int) -> bool:
        for i, t in enumerate(self.store.transactions):
            if t.id == t_id:
                del self.store.transactions[i]
                return True
        return False

    def update_goal(self, g_id: int, **kwargs) -> bool:
        for g in self.store.goals:
            if g.id == g_id:
                for k, v in kwargs.items():
                    if hasattr(g, k):
                        setattr(g, k, v)
                return True
        return False

    def delete_goal(self, g_id: int) -> bool:
        for i, g in enumerate(self.store.goals):
            if g.id == g_id:
                del self.store.goals[i]
                return True
        return False

    # Totals
    def total_income(self) -> float:
        return sum(t.amount for t in self.store.transactions if t.type == "INCOME")

    def total_expenses(self) -> float:
        return sum(t.amount for t in self.store.transactions if t.type == "EXPENSE")

    def net_savings(self) -> float:
        return self.total_income() - self.total_expenses()

    # Filtering / Searching
    def filter_expenses_over(self, threshold: float) -> List[Transaction]:
        return [t for t in self.store.transactions if t.type == "EXPENSE" and t.amount > threshold]

    def filter_by_category(self, category: str) -> List[Transaction]:
        return [t for t in self.store.transactions if t.category.lower() == category.lower()]

    def filter_by_date_range(self, start_date: str, end_date: str) -> List[Transaction]:
        # Assumes dates are in ISO format and comparable as strings
        return [
            t for t in self.store.transactions
            if start_date <= t.date <= end_date
        ]

    # Sorting
    def sort_transactions(self, key: str, reverse: bool = False) -> List[Transaction]:
        valid_keys = {"date", "amount", "category", "type", "id"}
        if key not in valid_keys:
            raise ValueError(f"Invalid sort key: {key}. Choose from {valid_keys}")
        return sorted(self.store.transactions, key=lambda t: getattr(t, key), reverse=reverse)

    # ASCII Chart: monthly expenses
    def monthly_expense_chart(self, year: int) -> str:
        # Build a dict: month -> total expenses in that month
        monthly: Dict[int, float] = {m: 0.0 for m in range(1, 13)}
        for t in self.store.transactions:
            if t.type == "EXPENSE":
                try:
                    dt = datetime.strptime(t.date, "%Y-%m-%d")
                except ValueError:
                    continue
                if dt.year == year:
                    monthly[dt.month] += t.amount

        max_expense = max(monthly.values()) if monthly else 0.0
        lines = []
        max_bar = 50  # characters
        lines.append(f"Monthly Expenses for {year}")
        for m in range(1, 13):
            value = monthly[m]
            if max_expense > 0:
                bar_len = int((value / max_expense) * max_bar)
            else:
                bar_len = 0
            bar = "#" * bar_len
            lines.append(f"{datetime(year, m, 1).strftime('%b')}: {bar} ({value:.2f})")
        return "\n".join(lines)

# -----------------------------
# Convenience / Presentation
# -----------------------------

def format_transaction(t: Transaction) -> str:
    return f"[{t.id}] {t.date} {t.type:<7} {t.category:<12} ${t.amount:>8.2f}  {t.note}"

def print_report(engine: FinanceEngine) -> None:
    print("=== Summary ===")
    print(f"Total Income:  ${engine.total_income():,.2f}")
    print(f"Total Expenses:${engine.total_expenses():,.2f}")
    print(f"Net Savings:   ${engine.net_savings():,.2f}")
    print()

    print("=== Expenses by Category (top 5) ===")
    exp_by_cat: Dict[str, float] = {}
    for t in engine.store.transactions:
        if t.type == "EXPENSE":
            exp_by_cat[t.category] = exp_by_cat.get(t.category, 0.0) + t.amount
    # sort by amount desc
    for cat, amt in sorted(exp_by_cat.items(), key=lambda kv: kv[1], reverse=True)[:5]:
        print(f"{cat:<15} ${amt:>10.2f}")
    print()

def list_transactions(engine: FinanceEngine, filter_func=None) -> None:
    txs = engine.store.transactions
    if filter_func:
        txs = list(filter(filter_func, txs))
    for t in txs:
        print(format_transaction(t))


if __name__ == "__main__":
    # Load existing data if present
    store = load_store()
    engine = FinanceEngine(store)

    
if not engine.store.transactions:
        engine.add_income("2025-01-15", "Salary", 5000, "January salary")
        engine.add_expense("2025-01-16", "Rent", 1500, "January rent")
        engine.add_expense("2025-01-17", "Groceries", 320, "Supermarket")
        engine.add_expense("2025-02-05", "Utilities", 180, "Electricity bill")
        engine.add_expense("2025-02-20", "Dining", 75, "Restaurant")
        engine.add_income("2025-02-28", "Freelance", 1200, "Project X")
        engine.add_goal("Emergency Fund", 5000, 1200, "2026-12-31")

    # Show some output
    print_report(engine)

    print("\n=== All Expenses Over $100 ===")
    for t in engine.filter_expenses_over(100):
        print(format_transaction(t))

    print("\n=== Monthly Expense Chart (Current Year) ===")
    current_year = datetime.now().year
    print(engine.monthly_expense_chart(current_year))

    # Save changes
    save_store(engine.store)
