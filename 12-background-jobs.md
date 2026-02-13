# 12 — Background Jobs & Active Job

---

## 12.1 Job Structure Convention

```ruby
# app/jobs/application_job.rb
class ApplicationJob < ActiveJob::Base
  # Retry on transient failures
  retry_on ActiveRecord::Deadlocked, wait: 5.seconds, attempts: 3
  retry_on Net::OpenTimeout, wait: :polynomially_longer, attempts: 10

  # Discard on permanent failures
  discard_on ActiveJob::DeserializationError

  # Global error handling
  rescue_from StandardError do |exception|
    ErrorTracker.capture(exception)
    raise  # Re-raise to trigger retry
  end
end
```

```ruby
# app/jobs/process_payment_job.rb
class ProcessPaymentJob < ApplicationJob
  queue_as :critical           # Queue name
  self.queue_adapter = :sidekiq # Override adapter per job (rare)

  # Retry configuration
  retry_on Stripe::RateLimitError, wait: 30.seconds, attempts: 5
  retry_on Stripe::APIConnectionError, wait: :polynomially_longer, attempts: 10
  discard_on Stripe::InvalidRequestError

  def perform(order_id)
    order = Order.find(order_id)
    return if order.paid?  # Idempotency guard

    PaymentService.charge(order)
    order.update!(status: :paid, paid_at: Time.current)
    OrderMailer.receipt(order).deliver_later
  end
end
```

---

## 12.2 Job Naming & Organization

```ruby
# Convention: <Verb><Noun>Job
SendWelcomeEmailJob           # app/jobs/send_welcome_email_job.rb
ProcessPaymentJob             # app/jobs/process_payment_job.rb
GenerateReportJob             # app/jobs/generate_report_job.rb
CleanupExpiredTokensJob       # app/jobs/cleanup_expired_tokens_job.rb
SyncInventoryJob              # app/jobs/sync_inventory_job.rb

# Namespaced jobs for organization
module Users
  class SendWelcomeEmailJob < ApplicationJob; end   # app/jobs/users/send_welcome_email_job.rb
  class CleanupInactiveJob < ApplicationJob; end     # app/jobs/users/cleanup_inactive_job.rb
end

module Reports
  class GenerateDailyJob < ApplicationJob; end       # app/jobs/reports/generate_daily_job.rb
end
```

---

## 12.3 Queue Conventions

```ruby
# Convention: Define queue priorities
# critical  — Payments, auth, time-sensitive operations
# default   — Standard processing
# low       — Reports, exports, cleanup
# mailers   — Email delivery

class ProcessPaymentJob < ApplicationJob
  queue_as :critical
end

class SendWelcomeEmailJob < ApplicationJob
  queue_as :mailers
end

class GenerateReportJob < ApplicationJob
  queue_as :low
end

# Dynamic queue assignment
class ExportJob < ApplicationJob
  queue_as do
    if self.arguments.first[:priority] == "high"
      :critical
    else
      :low
    end
  end
end
```

---

## 12.4 Enqueuing Jobs

```ruby
# Enqueue for immediate processing
ProcessPaymentJob.perform_later(order.id)

# Enqueue with delay
SendReminderJob.set(wait: 1.hour).perform_later(user.id)
SendReminderJob.set(wait_until: Date.tomorrow.noon).perform_later(user.id)

# Enqueue on specific queue
ExportJob.set(queue: :low).perform_later(report_params)

# Convention: ALWAYS pass IDs, not objects (serialization!)
ProcessPaymentJob.perform_later(order.id)    # ✅ Pass ID
ProcessPaymentJob.perform_later(order)       # ⚠️  Works but serializes via GlobalID

# Convention: Jobs should be idempotent (safe to run multiple times)
# Convention: Jobs should be small and focused (one concern per job)
# Convention: Use callbacks/service objects to enqueue jobs, not controllers directly
```

---

## 12.5 Idempotency Patterns

```ruby
class ProcessPaymentJob < ApplicationJob
  def perform(order_id)
    order = Order.find(order_id)

    # Guard clause — already processed
    return if order.paid?

    # Use database lock to prevent concurrent processing
    order.with_lock do
      return if order.paid?  # Double-check after lock
      PaymentService.charge(order)
      order.update!(status: :paid)
    end
  end
end

# Unique job (using sidekiq-unique-jobs or similar)
class SyncUserJob < ApplicationJob
  unique :until_executed  # Prevent duplicate jobs

  def perform(user_id)
    user = User.find(user_id)
    ExternalService.sync(user)
  end
end
```

---

## 12.6 Solid Queue (Rails 8 Default)

```ruby
# Rails 8 ships with Solid Queue — database-backed job queue
# config/queue.yml
default: &default
  dispatchers:
    - polling_interval: 1
      batch_size: 500
  workers:
    - queues: "*"
      threads: 3
      processes: 1
      polling_interval: 0.1

production:
  <<: *default
  workers:
    - queues: [critical, default]
      threads: 5
    - queues: [low, mailers]
      threads: 3

# config/application.rb
config.active_job.queue_adapter = :solid_queue
```

---

## 12.7 Recurring Jobs (Rails 8 / Solid Queue)

```ruby
# config/recurring.yml
production:
  cleanup_expired_sessions:
    class: CleanupExpiredSessionsJob
    schedule: every day at 3am
  
  generate_daily_report:
    class: Reports::GenerateDailyJob
    schedule: every day at 6am
  
  sync_inventory:
    class: SyncInventoryJob
    schedule: every 15 minutes
```
