# 📊 CloudWatch Monitoring & Alerting Dashboard

> **AWS Lab Project** | CloudWatch · Alarms · Dashboards · Metrics · Auto Scaling · SNS · Log Groups

---

## 📋 Project Overview

This project builds a complete CloudWatch monitoring setup for a web application running on EC2 with an Application Load Balancer and Auto Scaling Group. The goal: proactive visibility into system health so you catch problems before users do — and automate the response.

Includes custom metrics, multi-resource dashboards, CPU-based scale-out alarms, and ALB health tracking.

---

## 🏗️ Architecture Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                    CloudWatch Monitoring                      │
│                                                              │
│  DATA SOURCES                    ACTIONS                     │
│  ┌─────────────────┐             ┌──────────────────────┐    │
│  │  EC2 Instances  │──metrics──▶ │   CloudWatch Alarms  │    │
│  │  (CPU, network) │             │                      │    │
│  └─────────────────┘             │  CPU > 70% (2 min)   │──▶ Auto Scale OUT
│                                  │  CPU < 30% (5 min)   │──▶ Auto Scale IN
│  ┌─────────────────┐             │  ALB 5xx > 10 (1 min)│──▶ SNS Alert Email
│  │  ALB Metrics    │──metrics──▶ │  Healthy hosts < 2   │──▶ SNS Alert Email
│  │  (requests,5xx, │             └──────────────────────┘    │
│  │   latency)      │                                         │
│  └─────────────────┘             ┌──────────────────────┐    │
│                                  │  CloudWatch Dashboard │    │
│  ┌─────────────────┐             │                      │    │
│  │  Auto Scaling   │──metrics──▶ │  Widget 1: CPU       │    │
│  │  (in-service,   │             │  Widget 2: ALB reqs  │    │
│  │   scale events) │             │  Widget 3: ASG count │    │
│  └─────────────────┘             │  Widget 4: 5xx errors│    │
│                                  └──────────────────────┘    │
│  ┌─────────────────┐                                         │
│  │  CloudWatch     │             ┌──────────────────────┐    │
│  │  Logs (Apache,  │──logs────▶  │  Log Insights Queries│    │
│  │   app errors)   │             └──────────────────────┘    │
│  └─────────────────┘                                         │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔧 AWS Services Used

| Service | Purpose |
|---|---|
| **CloudWatch Metrics** | Collects CPU, network, disk, ALB, and ASG metrics automatically |
| **CloudWatch Alarms** | Triggers actions when metrics cross thresholds |
| **CloudWatch Dashboard** | Real-time visual overview of all key metrics |
| **CloudWatch Logs** | Captures application and system logs from EC2 instances |
| **CloudWatch Log Insights** | Query logs for errors, patterns, and trends |
| **SNS (Simple Notification Service)** | Sends email/SMS alerts when alarms trigger |
| **Auto Scaling** | Responds to CloudWatch alarms with scale-out/in actions |

---

## 🚀 Step-by-Step Build

### Step 1: Create SNS Topic for Alerts

```bash
# Create topic for alarm notifications
SNS_ARN=$(aws sns create-topic \
  --name "webapp-alerts" \
  --query 'TopicArn' \
  --output text)

# Subscribe your email
aws sns subscribe \
  --topic-arn $SNS_ARN \
  --protocol email \
  --notification-endpoint your-email@example.com

echo "SNS Topic: $SNS_ARN"
echo "Check your email and confirm the subscription!"
```

### Step 2: Create CPU Alarm → Scale Out

```bash
# When avg CPU > 70% for 2 consecutive minutes → trigger ASG scale-out policy
aws cloudwatch put-metric-alarm \
  --alarm-name "cpu-high-scale-out" \
  --alarm-description "Scale out when CPU exceeds 70% for 2 minutes" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=AutoScalingGroupName,Value=webapp-asg \
  --alarm-actions $SCALE_OUT_POLICY_ARN \
  --ok-actions $SNS_ARN \
  --treat-missing-data notBreaching

echo "✅ CPU high alarm created → will trigger scale-out"
```

### Step 3: Create CPU Alarm → Scale In

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "cpu-low-scale-in" \
  --alarm-description "Scale in when CPU drops below 30% for 5 minutes" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 60 \
  --evaluation-periods 5 \
  --threshold 30 \
  --comparison-operator LessThanThreshold \
  --dimensions Name=AutoScalingGroupName,Value=webapp-asg \
  --alarm-actions $SCALE_IN_POLICY_ARN \
  --treat-missing-data notBreaching

echo "✅ CPU low alarm created → will trigger scale-in"
```

### Step 4: Create ALB Health Alarm

```bash
# Alert when there are fewer than 2 healthy instances
aws cloudwatch put-metric-alarm \
  --alarm-name "alb-unhealthy-hosts" \
  --alarm-description "Alert if healthy host count drops below 2" \
  --metric-name HealthyHostCount \
  --namespace AWS/ApplicationELB \
  --statistic Average \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 2 \
  --comparison-operator LessThanThreshold \
  --dimensions \
    Name=LoadBalancer,Value=$ALB_ARN_SUFFIX \
    Name=TargetGroup,Value=$TG_ARN_SUFFIX \
  --alarm-actions $SNS_ARN

echo "✅ ALB health alarm created"
```

### Step 5: Create ALB 5xx Error Alarm

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "alb-5xx-errors" \
  --alarm-description "Alert on elevated server errors from ALB" \
  --metric-name HTTPCode_ELB_5XX_Count \
  --namespace AWS/ApplicationELB \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=LoadBalancer,Value=$ALB_ARN_SUFFIX \
  --alarm-actions $SNS_ARN \
  --treat-missing-data notBreaching

echo "✅ 5xx error alarm created"
```

### Step 6: Create CloudWatch Dashboard

```bash
# Apply the dashboard JSON (see configs/dashboard.json)
aws cloudwatch put-dashboard \
  --dashboard-name "webapp-monitoring" \
  --dashboard-body file://configs/dashboard.json

echo "✅ Dashboard created: webapp-monitoring"
echo "View at: https://console.aws.amazon.com/cloudwatch/home#dashboards:name=webapp-monitoring"
```

---

## 📄 Key CloudWatch Log Insights Queries

Run these in CloudWatch → Log Insights → Select your log group:

```sql
-- Find all HTTP 500 errors in Apache access log
fields @timestamp, @message
| filter @message like /HTTP\/1.1" 5/
| sort @timestamp desc
| limit 50

-- Count requests by status code
fields @timestamp, @message
| parse @message '"GET * HTTP/1.1" * *' as path, status, bytes
| stats count(*) as requests by status
| sort requests desc

-- Find slow requests (> 1 second response time)
fields @timestamp, @message
| filter @message like /\.[0-9]{3,}/
| sort @timestamp desc
| limit 20
```

---

## 📁 Repository Structure

```
07-cloudwatch-monitoring-dashboard/
├── README.md
├── configs/
│   └── dashboard.json              # CloudWatch dashboard definition
├── scripts/
│   ├── 01-create-sns-topic.sh
│   ├── 02-create-cpu-alarms.sh
│   ├── 03-create-alb-alarms.sh
│   ├── 04-create-dashboard.sh
│   └── 05-cloudwatch-agent-setup.sh  # For custom metrics from EC2
└── queries/
    └── log-insights-queries.sql    # Useful CloudWatch Log Insights queries
```

---

## 💡 Key Takeaways

1. **Evaluation periods matter** — 1 evaluation of 60 seconds is much more sensitive than 5 evaluations of 60 seconds. Tune based on how fast you want to react
2. **`treat-missing-data notBreaching`** — prevents false alarms when instances are just starting up and haven't reported yet
3. **Custom metrics require CloudWatch Agent** — default EC2 metrics don't include memory or disk; install the agent for those
4. **Dashboard JSON is version-controlled** — store it in Git so you can recreate it in any account
5. **Scale-out should be aggressive, scale-in conservative** — it's better to have one extra instance than to kill capacity during a traffic spike

---

## 🔗 Related Projects

- [02 - Highly Available Multi-Tier Web Application](../02-ha-multitier-webapp)
- [03 - Auto Scaling & Load Balancer](../03-auto-scaling-load-balancer)

---

*Built by Abdul Nazir Sadaat | AWS Cloud Engineer | Rancho Cordova, CA*
