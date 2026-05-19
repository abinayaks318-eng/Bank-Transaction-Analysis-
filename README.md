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

Below is the complete code structured according to the project layout specified in the report.

   Flask Application Factory (app/__init__.py)

from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_jwt_extended import JWTManager
from flask_bcrypt import Bcrypt
from config import Config

db  = SQLAlchemy()
jwt = JWTManager()
bcrypt = Bcrypt()

def create_app(config_class=Config):
    app = Flask(__name__)
    app.config.from_object(config_class)

    db.init_app(app)
    jwt.init_app(app)
    bcrypt.init_app(app)

    from app.routes.auth         import auth_bp
    from app.routes.transactions import txn_bp
    from app.routes.accounts     import acc_bp
    from app.routes.reports      import rep_bp

    app.register_blueprint(auth_bp, url_prefix='/api/auth')
    app.register_blueprint(txn_bp,  url_prefix='/api/transactions')
    app.register_blueprint(acc_bp,  url_prefix='/api/accounts')
    app.register_blueprint(rep_bp,  url_prefix='/api/reports')

    return app

     Transaction Processing Endpoint (routes/transactions.py)

from flask import Blueprint, request, jsonify
from flask_jwt_extended import jwt_required, get_jwt_identity
from app import db
from app.models import Account, Transaction
from app.services.fraud_service import run_fraud_check
from datetime import datetime
import uuid

txn_bp = Blueprint('transactions', __name__)

@txn_bp.route('/create', methods=['POST'])
@jwt_required()
def create_transaction():
    data = request.get_json()
    account_id   = data.get('account_id')
    txn_type     = data.get('transaction_type')   # CREDIT / DEBIT
    amount       = float(data.get('amount', 0))
    channel      = data.get('channel', 'ONLINE')
    description  = data.get('description', '')

    if amount <= 0:
        return jsonify({'error': 'Amount must be positive'}), 400

    account = Account.query.get_or_404(account_id)
    if account.status != 'ACTIVE':
        return jsonify({'error': 'Account is not active'}), 403

    if txn_type == 'DEBIT' and float(account.balance) < amount:
        return jsonify({'error': 'Insufficient funds'}), 400

    try:
        if txn_type == 'CREDIT':
            account.balance = float(account.balance) + amount
        else:
            account.balance = float(account.balance) - amount

        txn = Transaction(
            account_id       = account_id,
            transaction_type = txn_type,
            amount           = amount,
            balance_after    = account.balance,
            channel          = channel,
            description      = description,
            status           = 'SUCCESS',
            reference_no     = str(uuid.uuid4())[:20],
            completed_at     = datetime.utcnow()
        )
        db.session.add(txn)
        db.session.commit()

        # Run fraud detection asynchronously
        run_fraud_check(txn.transaction_id)

        return jsonify({
            'message'      : 'Transaction successful',
            'reference_no' : txn.reference_no,
            'balance_after': float(account.balance)
        }), 201

    except Exception as e:
        db.session.rollback()
        return jsonify({'error': str(e)}), 500
     Fraud Detection Service (services/.  fraud_service.py)
from app import db
from app.models import Transaction, Alert
from datetime import datetime, timedelta
import statistics

LARGE_AMOUNT_THRESHOLD = 100000   # 1 lakh INR
VELOCITY_LIMIT         = 5        # max transactions per 10 min
Z_SCORE_THRESHOLD      = 3.0

def run_fraud_check(transaction_id):
    txn = Transaction.query.get(transaction_id)
    if not txn:
        return

    alerts = []

    # Rule 1: Large amount check
    if float(txn.amount) > LARGE_AMOUNT_THRESHOLD:
        alerts.append(Alert(
            transaction_id = transaction_id,
            alert_type     = 'LARGE_AMOUNT',
            severity       = 'HIGH',
            description    = f'Transaction of {txn.amount} exceeds threshold'
        ))

    # Rule 2: Velocity check (> 5 txns in 10 minutes)
    window_start = txn.initiated_at - timedelta(minutes=10)
    recent_count = Transaction.query.filter(
        Transaction.account_id  == txn.account_id,
        Transaction.initiated_at >= window_start,
        Transaction.transaction_id != transaction_id
    ).count()

    if recent_count >= VELOCITY_LIMIT:
        alerts.append(Alert(
            transaction_id = transaction_id,
            alert_type     = 'HIGH_VELOCITY',
            severity       = 'MEDIUM',
            description    = f'{recent_count} transactions in last 10 minutes'
        ))

    # Rule 3: Z-score anomaly detection
    recent_txns = Transaction.query.filter(
        Transaction.account_id == txn.account_id,
        Transaction.status     == 'SUCCESS'
    ).order_by(Transaction.initiated_at.desc()).limit(30).all()

    if len(recent_txns) >= 5:
        amounts = [float(t.amount) for t in recent_txns]
        mean    = statistics.mean(amounts)
        stdev   = statistics.stdev(amounts)
        if stdev > 0:
            z = abs((float(txn.amount) - mean) / stdev)
            if z > Z_SCORE_THRESHOLD:
                alerts.append(Alert(
                    transaction_id = transaction_id,
                    alert_type     = 'STATISTICAL_ANOMALY',
                    severity       = 'HIGH',
                    description    = f'Z-score of {z:.2f} detected'
                ))

    for alert in alerts:
        db.session.add(alert)
    db.session.commit()

      Report Generation Endpoint (routes/reports.py)

from flask import Blueprint, request, jsonify
from flask_jwt_extended import jwt_required
from app.models import Transaction
from sqlalchemy import func
from datetime import datetime

rep_bp = Blueprint('reports', __name__)

@rep_bp.route('/summary', methods=['GET'])
@jwt_required()
def transaction_summary():
    account_id = request.args.get('account_id', type=int)
    start      = request.args.get('start_date')
    end        = request.args.get('end_date')

    query = Transaction.query.filter(
        Transaction.account_id == account_id,
        Transaction.status     == 'SUCCESS'
    )

    if start:
        query = query.filter(Transaction.initiated_at >= datetime.fromisoformat(start))
    if end:
        query = query.filter(Transaction.initiated_at <= datetime.fromisoformat(end))

    txns         = query.all()
    total_credit = sum(float(t.amount) for t in txns if t.transaction_type == 'CREDIT')
    total_debit  = sum(float(t.amount) for t in txns if t.transaction_type == 'DEBIT')
    txn_count    = len(txns)
    net_flow     = total_credit - total_debit
    avg_amount   = (total_credit + total_debit) / txn_count if txn_count else 0

    channel_summary = {}
    for t in txns:
        channel_summary[t.channel] = channel_summary.get(t.channel, 0) + 1

    return jsonify({
        'account_id'    : account_id,
        'period'        : { 'start': start, 'end': end },
        'total_credit'  : round(total_credit, 2),
        'total_debit'   : round(total_debit, 2),
        'net_flow'      : round(net_flow, 2),
        'txn_count'     : txn_count,
        'avg_amount'    : round(avg_amount, 2),
        'channel_summary': channel_summary
    }), 200

                Output

The following section illustrates sample API responses and system outputs produced during testing.

      Successful Transaction Response

POST /api/transactions/create
Authorization: Bearer <JWT_TOKEN>

Request Body:
{
  "account_id"       : 1042,
  "transaction_type" : "DEBIT",
  "amount"           : 25000.00,
  "channel"          : "ONLINE",
  "description"      : "Online Shopping - Amazon"
}

Response (201 Created):
{
  "message"       : "Transaction successful",
  "reference_no"  : "a3f2c891-4e0b-4f5d",
  "balance_after" : 75432.50
}

          Fraud Alert Response

Fraud Detection triggered for transaction_id: 8845

Alert Generated:
{
  "alert_id"      : 221,
  "transaction_id": 8845,
  "alert_type"    : "LARGE_AMOUNT",
  "severity"      : "HIGH",
  "description"   : "Transaction of 125000.00 exceeds threshold of 100000",
  "resolved"      : false,
  "created_at"    : "2024-11-15T14:32:10"
}

       Transaction Summary Report Output

GET /api/reports/summary?account_id=1042&start_date=2024-11-01&end_date=2024-11-30

Response (200 OK):
{
  "account_id"   : 1042,
  "period"       : { "start": "2024-11-01", "end": "2024-11-30" },
  "total_credit" : 150000.00,
  "total_debit"  : 92450.75,
  "net_flow"     : 57549.25,
  "txn_count"    : 38,
  "avg_amount"   : 6380.29,
  "channel_summary": {
    "ONLINE"  : 18,
    "UPI"     : 12,
    "ATM"     : 5,
    "BRANCH"  : 3
  }
}

                Conclusion :

The project demonstrates key software engineering principles including separation of concerns, code modularity, database normalization, RESTful API design, and error handling. It can serve as a foundation for a production-grade banking analytics platform with additional enhancements such as machine learning-based fraud detection, real-time streaming analytics using Apache Kafka, dashboard visualization using React.js, and multi-currency support.