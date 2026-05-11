# Alacio_tracker
So our Auntie' place has a problem with bad debts, so we trying to develop a tracker to keep the debts in check
-- models/marts/finance/customer_credit_risk.sql

WITH customer_payments AS (
    SELECT
        customer_id,
        SUM(amount_due)         AS total_credit_extended,
        SUM(amount_paid)        AS total_amount_paid,
        SUM(amount_overdue)     AS total_overdue,
        MAX(days_overdue)       AS max_days_overdue,
        COUNT(missed_payments)  AS missed_payment_count
    FROM {{ ref('stg_collections') }}
    GROUP BY customer_id
),

customer_risk_score AS (
    SELECT
        customer_id,
        total_credit_extended,
        total_overdue,
        missed_payment_count,
        max_days_overdue,

        -- Risk scoring logic agreed with finance team
        CASE
            WHEN max_days_overdue > 90
                OR missed_payment_count >= 3  THEN 'HIGH'
            WHEN max_days_overdue BETWEEN 30 AND 90
                OR missed_payment_count = 2   THEN 'MEDIUM'
            ELSE                                   'LOW'
        END AS risk_tier,

        -- Bad debt flag: >120 days overdue = write-off candidate
        CASE
            WHEN max_days_overdue > 120 THEN TRUE
            ELSE FALSE
        END AS bad_debt_flag

    FROM customer_payments
)

SELECT * FROM customer_risk_score
# Alert logic - runs daily after pipeline completes
def check_bad_debt_thresholds():
    
    high_risk_customers = query("""
        SELECT COUNT(*) as count, SUM(total_overdue) as exposure
        FROM customer_credit_risk
        WHERE risk_tier = 'HIGH'
        AND bad_debt_flag = TRUE
    """)
    
    # Alert finance team if exposure exceeds threshold
    if high_risk_customers['exposure'] > 50000:
        send_alert(
            to="finance-team@alacio.com",
            subject="⚠️ Bad Debt Alert: High Risk Exposure Detected",
            body=f"""
            {high_risk_customers['count']} customers flagged as high risk.
            Total exposure: ${high_risk_customers['exposure']:,.0f}
            Action required: Review collections dashboard immediately.
            """
        )


