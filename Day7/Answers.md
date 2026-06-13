# Part A Reflection Questions

## 1. Why do we put credentials in `.env` instead of `docker-compose.yaml` directly?

Credentials should be stored in a `.env` file rather than hardcoded in `docker-compose.yaml` to improve security, maintainability, and portability. Hardcoding secrets in configuration files increases the risk of exposing sensitive information through source control systems such as GitHub. By using environment variables, the same Docker Compose configuration can be reused across development, testing, and production environments without modifying application code. This follows the Twelve-Factor App methodology and industry best practices for secret management.

---

## 2. Why does the alert-service buffer and cool down alerts instead of sending one per event?

Alert buffering and cooldown mechanisms prevent alert fatigue. In a production environment, a single incident can generate thousands of repetitive events within a short period. Sending an email for every event would overwhelm engineers and make it difficult to identify critical issues. Buffering groups related alerts together, while cooldown periods suppress duplicate notifications for a predefined duration. This approach improves signal-to-noise ratio and ensures that engineers focus on actionable incidents rather than excessive notifications.

---

## 3. Why do warning emails and critical emails have different subject lines?

Different subject lines allow engineers and monitoring systems to quickly classify the severity of an incident. Critical alerts typically require immediate attention because they indicate service degradation, outages, or potential business impact. Warning alerts indicate abnormal behavior that should be investigated but may not require urgent intervention. Distinct subject lines improve triage efficiency, enable automated email filtering, and help on-call engineers prioritize responses according to incident severity.

---

## 4. If you wanted to send Slack alerts instead of email, which file would you change?

The alert delivery logic is implemented in the alert service. To replace email notifications with Slack notifications, I would modify the alert-service application code responsible for sending alerts. Instead of invoking AWS SES APIs, the service would send HTTP requests to a Slack Incoming Webhook URL or use the Slack API SDK. The monitoring and alert generation logic would remain unchanged; only the notification delivery layer would be replaced.

---

## 5. Why is Docker Compose not used in production for this kind of app?

Docker Compose is designed primarily for local development, testing, and small-scale deployments. It lacks advanced production features such as automatic scaling, self-healing across multiple hosts, rolling deployments, service discovery, load balancing, and enterprise-grade orchestration. Production environments typically use container orchestration platforms such as Kubernetes, Amazon ECS, or Docker Swarm, which provide high availability, fault tolerance, centralized management, and automated recovery. Docker Compose is excellent for development workflows, but production systems require orchestration platforms that can manage containers reliably at scale.

