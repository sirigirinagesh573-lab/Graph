# Graph
 Student expenses 
import streamlit as st
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime

# ==========================================
# 1. THE EVENTS (Architectural Structures)
# ==========================================
class BudgetSetEvent:
    def __init__(self, amount: float):
        self.type = "BUDGET_SET"
        self.amount = amount
        self.timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

class IncomeRecordedEvent:
    def __init__(self, source: str, amount: float):
        self.type = "INCOME"
        self.source = source
        self.amount = amount
        self.timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

class ExpenseRecordedEvent:
    def __init__(self, category: str, description: str, amount: float):
        self.type = "EXPENSE"
        self.category = category
        self.description = description
        self.amount = amount
        self.timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

# ==========================================
# 2. STATE APP ENGINE (Event Processing)
# ==========================================
def rebuild_state(event_log):
    """Processes the historical event log to generate the active UI state."""
    state = {
        "budget": 0.0,
        "total_income": 0.0,
        "total_expense": 0.0,
        "categories": {"Food": 0.0, "Travel": 0.0, "Personal Needs": 0.0, "Other": 0.0},
        "history": []
    }
    
    for event in event_log:
        if event.type == "BUDGET_SET":
            state["budget"] = event.amount
        elif event.type == "INCOME":
            state["total_income"] += event.amount
            state["history"].append({"Time": event.timestamp, "Type": "💰 Income", "Details": event.source, "Amount": event.amount})
        elif event.type == "EXPENSE":
            state["total_expense"] += event.amount
            state["categories"][event.category] = state["categories"].get(event.category, 0.0) + event.amount
            state["history"].append({"Time": event.timestamp, "Type": "💸 Expense", "Details": f"[{event.category}] {event.description}", "Amount": -event.amount})
            
    return state

# ==========================================
# 3. STREAMLIT USER INTERFACE
# ==========================================
st.set_page_config(page_title="Student Expense Tracker", layout="wide", page_icon="💳")

# Keep the Event Log persistent across page reruns
if "event_log" not in st.session_state:
    st.session_state.event_log = [
        # Pre-populate sample events for context
        BudgetSetEvent(amount=10000.00),
        IncomeRecordedEvent(source="Monthly Allowance", amount=12000.00),
        ExpenseRecordedEvent(category="Food", description="Mess Fee", amount=4000.00),
        ExpenseRecordedEvent(category="Travel", description="Train Ticket", amount=1500.00)
    ]

# Calculate fresh state from the source of truth (the event log)
app_state = rebuild_state(st.session_state.event_log)

st.title("💳 Student Event-Driven Expense Tracker")
st.markdown("Track your money, understand your habits, and never run out of cash before month-end.")

# ----------------- SYSTEM NOTIFICATIONS / ALERTS -----------------
if app_state["budget"] > 0:
    usage_pct = (app_state["total_expense"] / app_state["budget"]) * 100
    if app_state["total_expense"] > app_state["budget"]:
        st.error(f"🚨 **Budget Blown!** You have exceeded your limit by **₹{app_state['total_expense'] - app_state['budget']:.2f}**.")
    elif usage_pct >= 80.0:
        st.warning(f"⚠️ **Warning:** You have used **{usage_pct:.1f}%** of your target monthly budget! (Remaining: ₹{app_state['budget'] - app_state['total_expense']:.2f})")

# ----------------- MAIN METRICS DASHBOARD -----------------
col1, col2, col3, col4 = st.columns(4)
col1.metric("Target Monthly Budget", f"₹{app_state['budget']:.2f}")
col2.metric("Total Income Added", f"₹{app_state['total_income']:.2f}")
col3.metric("Total Money Spent", f"₹{app_state['total_expense']:.2f}")
balance = app_state["total_income"] - app_state["total_expense"]
col4.metric("Available Balance", f"₹{balance:.2f}", delta=f"₹{balance}" if balance >= 0 else f"-₹{abs(balance)}")

st.divider()

# ----------------- ACTION INPUT CONTROLS -----------------
st.subheader("➕ Record a New Action (Fire Event)")
tab1, tab2, tab3 = st.tabs(["Set Target Budget", "Log Income", "Log Expense"])

with tab1:
    with st.form("budget_form", clear_on_submit=True):
        b_amount = st.number_input("Enter Monthly Spending Cap (₹)", min_value=0.0, step=500.0)
        if st.form_submit_button("Update Budget Limit"):
            st.session_state.event_log.append(BudgetSetEvent(b_amount))
            st.rerun()

with tab2:
    with st.form("income_form", clear_on_submit=True):
        i_source = st.text_input("Source", placeholder="e.g., Part-time job, Pocket money")
        i_amount = st.number_input("Amount Received (₹)", min_value=0.0, step=100.0)
        if st.form_submit_button("Record Income Entry"):
            if i_source and i_amount > 0:
                st.session_state.event_log.append(IncomeRecordedEvent(i_source, i_amount))
                st.rerun()
            else:
                st.error("Please provide valid inputs.")

with tab3:
    with st.form("expense_form", clear_on_submit=True):
        e_cat = st.selectbox("Category", ["Food", "Travel", "Personal Needs", "Other"])
        e_desc = st.text_input("Description / Notes", placeholder="e.g., Grocery shopping, Bus fare")
        e_amount = st.number_input("Amount Spent (₹)", min_value=0.0, step=50.0)
        if st.form_submit_button("Record Expense Entry"):
            if e_desc and e_amount > 0:
                st.session_state.event_log.append(ExpenseRecordedEvent(e_cat, e_desc, e_amount))
                st.rerun()
            else:
                st.error("Please provide valid inputs.")

st.divider()

# ----------------- VISUAL ANALYTICS & DATA -----------------
left_col, right_col = st.columns([1, 1])

with left_col:
    st.subheader("📊 Category Summaries")
    cat_data = pd.DataFrame(list(app_state["categories"].items()), columns=["Category", "Amount Spent"])
    
    if app_state["total_expense"] > 0:
        fig, ax = plt.subplots(figsize=(6, 4))
        # Filter out zero categories to keep the visual neat
        vis_data = cat_data[cat_data["Amount Spent"] > 0]
        ax.pie(vis_data["Amount Spent"], labels=vis_data["Category"], autopct='%1.1f%%', startangle=90, colors=['#ff9999','#6bbc6b','#99ff99','#ffcc99'])
        ax.axis('equal')
        st.pyplot(fig)
    else:
        st.info("No expense events recorded yet to render charts.")

with right_col:
    st.subheader("📜 Event Audit Log")
    if app_state["history"]:
        df = pd.DataFrame(app_state["history"])
        # Display the transactions ledger backward (newest first)
        st.dataframe(df.iloc[::-1], use_container_width=True, hide_index=True)
    else:
        st.text("Your financial history ledger is completely empty."
