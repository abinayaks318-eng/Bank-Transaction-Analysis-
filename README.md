BANK TRANSACTION ANALYSIS

          📋 Project Overview
The document is a comprehensive project report for a Bank Transaction Analysis System (BTAS) developed for the academic year 2024 - 2025.  

        Introduction & Objectives

Purpose: A software solution designed to automate the collection, processing, and analysis of large volumes of digital banking transactions. It replaces manual data monitoring to provide real-time insights, flag anomalies, and ensure regulatory compliance.  

Core Stack: Python (Flask) for the backend REST API, MySQL as the relational database, and SQLAlchemy ORM.  

             Report Generation: 

Aggregates transaction data within specified dates to compute total credits/debits, net flows, transaction counts, and channel-wise summaries.  

               Primary Focus:

Secure database storage, CRUD operations via an API, financial metric computation, rule-based fraud detection, and audit report generation. 

    Problem Statement & Proposed Solution

The Problem: Manual monitoring cannot scale with modern transaction volumes, leading to delayed fraud detection, data dispersion across disjointed systems, and difficulties meeting AML (Anti-Money Laundering) and KYC regulations.  



The Solution: A centralized, automated platform that processes transactions atomically, applies rule-based/statistical fraud detection checks in real time, and exposes secure endpoints for consolidated auditing.  


                 Data Model

Schema Design: Implements a normalized relational database schema (3NF).  



Entities: Core tables include Customer, Account, Transaction, Branch, Users, and Alert.  

             Algorithmic Logic

Transaction Processing: Authenticates requests via JWT, verifies account state, safely computes new balances under an atomic database transaction, commits changes, and triggers asynchronous fraud checks.  

           Fraud Detection Engine:

Implements a 5-rule check evaluating large amount thresholds (> 100k INR), velocity checks (> 5 transactions in 10 mins), statistical anomalies (Z-score > 3.0), off-hours processing, and geographic mismatches.  

           SPENDING BY CATEGORIES

Travel dominates total spend at 26.6%, followed by Shopping (20.0%) and ATM withdrawals (18.8%). These three categories account for nearly two-thirds of all expenditure.

Category      | Total Spend  | Avg/Txn  | Share % | Txns
--------------|--------------|----------|---------|-----
Travel        | Rs. 4,66,604 | Rs. 8,804| 26.6%   | 53
Shopping      | Rs. 3,51,101 | Rs. 7,165| 20.0%   | 49
ATM           | Rs. 3,28,882 | Rs. 6,852| 18.8%   | 48
Education     | Rs. 1,55,850 | Rs. 2,514|  8.9%   | 62
Dining        | Rs. 1,25,920 | Rs. 2,422|  7.2%   | 52
Utilities     | Rs. 1,01,287 | Rs. 1,688|  5.8%   | 60
Healthcare    | Rs. 87,167   | Rs. 2,027|  5.0%   | 43
Groceries     | Rs. 85,115   | Rs. 2,027|  4.8%   | 42
Entertainment | Rs. 34,902   | Rs. 793  |  2.0%   | 44
Transport     | Rs. 17,410   | Rs. 405  |  1.0%   | 43

Detection Methods:
[1] Z-Score Analysis
    Flags transactions whose amount exceeds 2.5 standard deviations from the mean across all transactions.
[2] Duplicate Detection
    Identifies rows sharing the same amount, merchant, and date on the same day.
[3] Large Round Amount
    Flags amounts >= Rs. 10,000 that are rounded to the nearest Rs. 1,000 (potential cash stuffing).

Flagged Transactions:
TXN ID    | Date       | Amount        | Z-Score | Reason
----------|------------|---------------|---------|-------------------
TXN00478  | 2024-10-22 | Rs. 80,024.80  | 10.58   | Extremely high value
TXN00341  | 2024-12-13 | Rs. 70,794.68  |  9.30   | Extremely high value
TXN00087  | 2024-01-27 | Rs. 69,683.81  |  9.15   | Extremely high value
TXN00412  | 2024-03-17 | Rs. 63,891.54  |  8.35   | Extremely high value
TXN00203  | 2024-05-27 | Rs. 62,491.29  |  8.16   | Extremely high value

MODULE REFERENCE
The project is organized into six focused modules under src/, each with a single responsibility.

Module         | File                | Responsibility
---------------|---------------------|------------------------------------
Data Loader    | data_loader.py      | Reads and validates CSV files; checks required columns
Cleaner        | cleaner.py          | Parses dates, removes duplicates, normalizes strings, adds derived cols
Analyzer       | analyzer.py         | Computes monthly summaries, category breakdowns, top merchants, statistics
Visualizer     | visualizer.py       | Generates 6 matplotlib/seaborn charts saved to reports/
Anomaly Det.   | anomaly_detector.py | Z-score, duplicate, and large-round-amount detection
Report Gen.    | report_generator.py | Builds formatted text reports; exports cleaned CSV

         Extracted Code Modules

Below is the complete code structured according to the project layout specified in the report.# ==============================================================================
# 1. DATABASE MODELS (models.py) [cite: 91, 92]
# ==============================================================================
from app import db [cite: 92]
from datetime import datetime [cite: 92]
import uuid [cite: 92]

class Customer(db.Model): [cite: 92]
    __tablename__ = 'customer' [cite: 92]
    customer_id  = db.Column(db.Integer, primary_key=True) [cite: 92]
    full_name    = db.Column(db.String(100), nullable=False) [cite: 92]
    email        = db.Column(db.String(100), unique=True, nullable=False) [cite: 92]
    phone        = db.Column(db.String(15), nullable=False) [cite: 92]
    pan_number   = db.Column(db.String(10), unique=True, nullable=False) [cite: 92]
    kyc_verified = db.Column(db.Boolean, default=False) [cite: 92]
    accounts     = db.relationship('Account', backref='owner', lazy=True) [cite: 92]

class Account(db.Model): [cite: 92]
    __tablename__ = 'account' [cite: 92]
    account_id     = db.Column(db.Integer, primary_key=True) [cite: 92]
    account_number = db.Column(db.String(20), unique=True, nullable=False) [cite: 92]
    account_type   = db.Column(db.Enum('SAVINGS','CURRENT','FIXED'), nullable=False) [cite: 92]
    balance        = db.Column(db.Numeric(15,2), default=0.00) [cite: 92]
    status         = db.Column(db.Enum('ACTIVE','INACTIVE','FROZEN'), default='ACTIVE') [cite: 92]
    customer_id    = db.Column(db.Integer, db.ForeignKey('customer.customer_id')) [cite: 92]
    transactions   = db.relationship('Transaction', backref='account', lazy=True) [cite: 92]

class Transaction(db.Model): [cite: 92]
    __tablename__ = 'transaction' [cite: 92]
    transaction_id   = db.Column(db.Integer, primary_key=True) [cite: 92]
    account_id       = db.Column(db.Integer, db.ForeignKey('account.account_id')) [cite: 92]
    transaction_type = db.Column(db.Enum('CREDIT','DEBIT','TRANSFER'), nullable=False) [cite: 92]
    amount           = db.Column(db.Numeric(15,2), nullable=False) [cite: 92]
    balance_after    = db.Column(db.Numeric(15,2), nullable=False) [cite: 92]
    channel          = db.Column(db.String(20)) [cite: 92]
    status           = db.Column(db.Enum('SUCCESS','FAILED','PENDING'), default='PENDING') [cite: 92]
    reference_no     = db.Column(db.String(30), unique=True, [cite: 92]
                                 default=lambda: str(uuid.uuid4())[:20]) [cite: 92]
    initiated_at     = db.Column(db.DateTime, default=datetime.utcnow) [cite: 92]


# ==============================================================================
# 2. FLASK APPLICATION FACTORY (app/__init__.py) [cite: 94, 95]
# ==============================================================================
from flask import Flask [cite: 95]
from flask_sqlalchemy import SQLAlchemy [cite: 95]
from flask_jwt_extended import JWTManager [cite: 95]
from flask_bcrypt import Bcrypt [cite: 95]
from config import Config [cite: 95]

db  = SQLAlchemy() [cite: 95]
jwt = JWTManager() [cite: 95]
bcrypt = Bcrypt() [cite: 95]

def create_app(config_class=Config): [cite: 95]
    app = Flask(__name__) [cite: 95]
    app.config.from_object(config_class) [cite: 95]

    db.init_app(app) [cite: 95]
    jwt.init_app(app) [cite: 95]
    bcrypt.init_app(app) [cite: 95]

    from app.routes.auth         import auth_bp [cite: 95]
    from app.routes.transactions import txn_bp [cite: 95]
    from app.routes.accounts     import acc_bp [cite: 95]
    from app.routes.reports      import rep_bp [cite: 95]

    app.register_blueprint(auth_bp, url_prefix='/api/auth') [cite: 95]
    app.register_blueprint(txn_bp,  url_prefix='/api/transactions') [cite: 95]
    app.register_blueprint(acc_bp,  url_prefix='/api/accounts') [cite: 95]
    app.register_blueprint(rep_bp,  url_prefix='/api/reports') [cite: 95]

    return app [cite: 95]


# ==============================================================================
# 3. TRANSACTION PROCESSING ENDPOINT (routes/transactions.py) [cite: 96, 97]
# ==============================================================================
from flask import Blueprint, request, jsonify [cite: 97]
from flask_jwt_extended import jwt_required, get_jwt_identity [cite: 97]
from app import db [cite: 97]
from app.models import Account, Transaction [cite: 97]
from app.services.fraud_service import run_fraud_check [cite: 97]
from datetime import datetime [cite: 97]
import uuid [cite: 97]

txn_bp = Blueprint('transactions', __name__) [cite: 97]

@txn_bp.route('/create', methods=['POST']) [cite: 97]
@jwt_required() [cite: 97]
def create_transaction(): [cite: 97]
    data = request.get_json() [cite: 97]
    account_id   = data.get('account_id') [cite: 97]
    txn_type     = data.get('transaction_type')   # CREDIT / DEBIT [cite: 97]
    amount       = float(data.get('amount', 0)) [cite: 97]
    channel      = data.get('channel', 'ONLINE') [cite: 97]
    description  = data.get('description', '') [cite: 97]

    if amount <= 0: [cite: 97]
        return jsonify({'error': 'Amount must be positive'}), 400 [cite: 97]

    account = Account.query.get_or_404(account_id) [cite: 97]
    if account.status != 'ACTIVE': [cite: 97]
        return jsonify({'error': 'Account is not active'}), 403 [cite: 97]

    if txn_type == 'DEBIT' and float(account.balance) < amount: [cite: 97]
        return jsonify({'error': 'Insufficient funds'}), 400 [cite: 97]

    try: [cite: 97]
        if txn_type == 'CREDIT': [cite: 97]
            account.balance = float(account.balance) + amount [cite: 97]
        else: [cite: 97]
            account.balance = float(account.balance) - amount [cite: 97]

        txn = Transaction( [cite: 97]
            account_id       = account_id, [cite: 97]
            transaction_type = txn_type, [cite: 97]
            amount           = amount, [cite: 97]
            balance_after    = account.balance, [cite: 97]
            channel          = channel, [cite: 97]
            description      = description, [cite: 97]
            status           = 'SUCCESS', [cite: 97]
            reference_no     = str(uuid.uuid4())[:20], [cite: 97]
            completed_at     = datetime.utcnow(), [cite: 97]
        ) [cite: 97]
        db.session.add(txn) [cite: 97]
        db.session.commit() [cite: 97]

        # Run fraud detection asynchronously [cite: 97]
        run_fraud_check(txn.transaction_id) [cite: 97]

        return jsonify({ [cite: 97]
            'message'      : 'Transaction successful', [cite: 97]
            'reference_no' : txn.reference_no, [cite: 97]
            'balance_after': float(account.balance) [cite: 97]
        }), 201 [cite: 97]

    except Exception as e: [cite: 97]
        db.session.rollback() [cite: 97]
        return jsonify({'error': str(e)}), 500 [cite: 97]


# ==============================================================================
# 4. FRAUD DETECTION SERVICE (services/fraud_service.py) [cite: 98, 99]
# ==============================================================================
from app import db [cite: 99]
from app.models import Transaction, Alert [cite: 99]
from datetime import datetime, timedelta [cite: 99]
import statistics [cite: 99]

LARGE_AMOUNT_THRESHOLD = 100000   # 1 lakh INR [cite: 99]
VELOCITY_LIMIT         = 5        # max transactions per 10 min [cite: 99]
Z_SCORE_THRESHOLD      = 3.0 [cite: 99]

def run_fraud_check(transaction_id): [cite: 99]
    txn = Transaction.query.get(transaction_id) [cite: 99]
    if not txn: [cite: 99]
        return [cite: 99]

    alerts = [] [cite: 99]

    # Rule 1: Large amount check [cite: 99]
    if float(txn.amount) > LARGE_AMOUNT_THRESHOLD: [cite: 99]
        alerts.append(Alert( [cite: 99]
            transaction_id = transaction_id, [cite: 99]
            alert_type     = 'LARGE_AMOUNT', [cite: 99]
            severity       = 'HIGH', [cite: 99]
            description    = f'Transaction of {txn.amount} exceeds threshold', [cite: 99]
        )) [cite: 99]

    # Rule 2: Velocity check (> 5 txns in 10 minutes) [cite: 99]
    window_start = txn.initiated_at - timedelta(minutes=10) [cite: 99]
    recent_count = Transaction.query.filter( [cite: 99]
        Transaction.account_id  == txn.account_id, [cite: 99]
        Transaction.initiated_at >= window_start, [cite: 99]
        Transaction.transaction_id != transaction_id, [cite: 99]
    ).count() [cite: 99]

    if recent_count >= VELOCITY_LIMIT: [cite: 99]
        alerts.append(Alert( [cite: 99]
            transaction_id = transaction_id, [cite: 99]
            alert_type     = 'HIGH_VELOCITY', [cite: 99]
            severity       = 'MEDIUM', [cite: 99]
            description    = f'{recent_count} transactions in last 10 minutes', [cite: 99]
        )) [cite: 99]

    # Rule 3: Z-score anomaly detection [cite: 99]
    recent_txns = Transaction.query.filter( [cite: 99]
        Transaction.account_id == txn.account_id, [cite: 99]
        Transaction.status     == 'SUCCESS', [cite: 99]
    ).order_by(Transaction.initiated_at.desc()).limit(30).all() [cite: 99]

    if len(recent_txns) >= 5: [cite: 99]
        amounts = [float(t.amount) for t in recent_txns] [cite: 99]
        mean    = statistics.mean(amounts) [cite: 99]
        stdev   = statistics.stdev(amounts) [cite: 99]
        if stdev > 0: [cite: 99]
            z = abs((float(txn.amount) - mean) / stdev) [cite: 99]
            if z > Z_SCORE_THRESHOLD: [cite: 99]
                alerts.append(Alert( [cite: 99]
                    transaction_id = transaction_id, [cite: 99]
                    alert_type     = 'STATISTICAL_ANOMALY', [cite: 99]
                    severity       = 'HIGH', [cite: 99]
                    description    = f'Z-score of {z:.2f} detected', [cite: 99]
                )) [cite: 99]

    for alert in alerts: [cite: 99]
        db.session.add(alert) [cite: 99]
    db.session.commit() [cite: 99]


# ==============================================================================
# 5. REPORT GENERATION ENDPOINT (routes/reports.py) [cite: 100, 101]
# ==============================================================================
from flask import Blueprint, request, jsonify [cite: 101]
from flask_jwt_extended import jwt_required [cite: 101]
from app.models import Transaction [cite: 101]
from sqlalchemy import func [cite: 101]
from datetime import datetime [cite: 101]

rep_bp = Blueprint('reports', __name__) [cite: 101]

@rep_bp.route('/summary', methods=['GET']) [cite: 101]
@jwt_required() [cite: 101]
def transaction_summary(): [cite: 101]
    account_id = request.args.get('account_id', type=int) [cite: 101]
    start      = request.args.get('start_date') [cite: 101]
    end        = request.args.get('end_date') [cite: 101]

    query = Transaction.query.filter( [cite: 101]
        Transaction.account_id == account_id, [cite: 101]
        Transaction.status     == 'SUCCESS', [cite: 101]
    ) [cite: 101]

    if start: [cite: 101]
        query = query.filter(Transaction.initiated_at >= datetime.fromisoformat(start)) [cite: 101]
    if end: [cite: 101]
        query = query.filter(Transaction.initiated_at <= datetime.fromisoformat(end)) [cite: 101]

    txns         = query.all() [cite: 101]
    total_credit = sum(float(t.amount) for t in txns if t.transaction_type == 'CREDIT') [cite: 101]
    total_debit  = sum(float(t.amount) for t in txns if t.transaction_type == 'DEBIT') [cite: 101]
    txn_count    = len(txns) [cite: 101]
    net_flow     = total_credit - total_debit [cite: 101]
    avg_amount   = (total_credit + total_debit) / txn_count if txn_count else 0 [cite: 101]

    channel_summary = {} [cite: 101]
    for t in txns: [cite: 101]
        channel_summary[t.channel] = channel_summary.get(t.channel, 0) + 1 [cite: 101]

    return jsonify({ [cite: 101]
        'account_id'    : account_id, [cite: 101]
        'period'        : { 'start': start, 'end': end }, [cite: 101]
        'total_credit'  : round(total_credit, 2), [cite: 101]
        'total_debit'   : round(total_debit, 2), [cite: 101]
        'net_flow'      : round(net_flow, 2), [cite: 101]
        'txn_count'     : txn_count, [cite: 101]
        'avg_amount'    : round(avg_amount, 2), [cite: 101]
        'channel_summary': channel_summary [cite: 101]
    }), 200 [cite: 101]  
