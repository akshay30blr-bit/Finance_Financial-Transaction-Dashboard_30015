# app.py
import streamlit as st
import pandas as pd
from database_ops import (
    create_db_and_table,
    fetch_data,
    insert_transaction,
    update_transaction,
    delete_transaction,
    get_aggregates
)
from datetime import datetime

# Initialize the database and table
create_db_and_table()

st.title("Financial Transaction Dashboard")

# UI Layout using tabs for a cleaner design
tab1, tab2 = st.tabs(["Dashboard", "Manage Transactions"])

with tab1:
    # --- Overview & Metrics (R) ---
    st.header("Financial Overview")
    total_transactions, total_revenue, total_expense, net_income = get_aggregates()

    col1, col2, col3, col4 = st.columns(4)
    with col1:
        st.metric(label="Total Transactions", value=f"{total_transactions}")
    with col2:
        st.metric(label="Total Revenue", value=f"${total_revenue:,.2f}")
    with col3:
        st.metric(label="Total Expenses", value=f"${total_expense:,.2f}")
    with col4:
        st.metric(label="Net Income", value=f"${net_income:,.2f}", delta=f"${net_income:,.2f}")
    
    st.markdown("---")
    
    # --- Transaction Details & Filtering (R) ---
    st.header("Transaction History")
    
    col_filter, col_sort, col_order = st.columns([1.5, 1, 1])
    with col_filter:
        transaction_type = st.selectbox(
            "Filter by Transaction Type", 
            options=["All", "Revenue", "Expense"],
            key="filter_select"
        )
    with col_sort:
        sort_by = st.selectbox(
            "Sort by", 
            options=['transaction_date', 'amount'],
            key="sort_select"
        )
    with col_order:
        sort_order = st.radio(
            "Sort Order", 
            options=['Ascending', 'Descending'],
            key="order_radio"
        )
    
    query = "SELECT * FROM transactions"
    params = None
    if transaction_type != "All":
        query += " WHERE type = %s"
        params = (transaction_type,)
    
    query += f" ORDER BY {sort_by}"
    if sort_order == 'Descending':
        query += " DESC"
    else:
        query += " ASC"

    filtered_df = fetch_data(query, params)
    
    if not filtered_df.empty:
        # Convert date to datetime object for proper display
        filtered_df['transaction_date'] = pd.to_datetime(filtered_df['transaction_date'])
        st.dataframe(filtered_df)
    else:
        st.info("No transactions found matching the criteria.")

with tab2:
    st.header("Manage Transactions")
    
    # --- Add New Transaction (C) ---
    st.subheader("Add a New Transaction")
    with st.form(key='add_transaction_form'):
        add_transaction_id = st.text_input("Transaction ID", help="Must be a unique ID, e.g., T007")
        add_transaction_date = st.date_input("Transaction Date", value=datetime.now())
        add_description = st.text_area("Description")
        add_amount = st.number_input("Amount", min_value=0.01, format="%.2f")
        add_type = st.selectbox("Type", ["Revenue", "Expense"])
        
        add_button = st.form_submit_button(label='Add Transaction')

        if add_button:
            if not add_transaction_id:
                st.error("Transaction ID is required.")
            else:
                data_to_insert = (
                    add_transaction_id, 
                    add_transaction_date.strftime('%Y-%m-%d'), 
                    add_description, 
                    add_amount, 
                    add_type
                )
                if insert_transaction(data_to_insert):
                    st.success("Transaction added successfully!")
                    st.experimental_rerun()
                else:
                    st.error("Failed to add transaction. ID might already exist.")

    st.markdown("---")

    # --- Update & Delete Transactions (U & D) ---
    st.subheader("Update or Delete an Existing Transaction")

    # Get a list of all existing transaction IDs for the selectbox
    all_ids_df = fetch_data("SELECT transaction_id FROM transactions ORDER BY transaction_id;")
    all_ids = all_ids_df['transaction_id'].tolist() if not all_ids_df.empty else []

    if not all_ids:
        st.info("No transactions to update or delete.")
    else:
        # Select transaction to edit/delete
        selected_id = st.selectbox(
            "Select a Transaction ID to Edit or Delete", 
            options=[""] + all_ids,
            key="update_select_id"
        )
        
        if selected_id:
            # Fetch the selected transaction's data to pre-populate form
            selected_df = fetch_data("SELECT * FROM transactions WHERE transaction_id = %s", (selected_id,))
            if not selected_df.empty:
                selected_data = selected_df.iloc[0]
                
                with st.form(key='update_transaction_form'):
                    st.write(f"Editing Transaction ID: **{selected_id}**")
                    update_date = st.date_input(
                        "Transaction Date", 
                        value=datetime.strptime(str(selected_data['transaction_date']), '%Y-%m-%d').date()
                    )
                    update_description = st.text_area("Description", value=selected_data['description'])
                    update_amount = st.number_input("Amount", value=float(selected_data['amount']), min_value=0.01, format="%.2f")
                    update_type = st.selectbox(
                        "Type", 
                        ["Revenue", "Expense"], 
                        index=["Revenue", "Expense"].index(selected_data['type'])
                    )
                    
                    col_update, col_delete = st.columns(2)
                    with col_update:
                        update_button = st.form_submit_button(label='Update Transaction')
                    with col_delete:
                        delete_button = st.form_submit_button(label='Delete Transaction')

                    if update_button:
                        data_to_update = (
                            update_date.strftime('%Y-%m-%d'),
                            update_description,
                            update_amount,
                            update_type
                        )
                        if update_transaction(selected_id, data_to_update):
                            st.success(f"Transaction {selected_id} updated successfully!")
                            st.experimental_rerun()
                        else:
                            st.error(f"Failed to update transaction {selected_id}.")
                    
                    if delete_button:
                        if delete_transaction(selected_id):
                            st.success(f"Transaction {selected_id} deleted successfully!")
                            st.experimental_rerun()
                        else:
                            st.error(f"Failed to delete transaction {selected_id}.")
# Finance_Financial-Transaction-Dashboard_30015
