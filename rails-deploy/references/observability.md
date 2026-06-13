# Observability — Logs, Error Tracking, Metrics, APM & Uptime

The merged observability topic, distinctly invocable. **You can't operate what you
can't see.** Four pillars: **structured logs**, **error tracking** (the menu),
**metrics/APM**, and **uptime** checks. Wire these as part of going to production —
not after the first incident.

## Error tracking (the menu) → Sentry / Honeybadger / AppSignal

```ruby
# Sentry (Recommended)
# Gemfile: gem "sentry-ruby"; gem "sentry-rails"
# config/initializers/sentry.rb
Sentry.init do |config|
  config.dsn = Rails.application.credentials.dig(:sentry, :dsn)   # secret → ../rails-security/
  config.traces_sample_rate = 0.2                                  # performance traces
  config.environment = Rails.env
end
```

| Option | Pick when |
|---|---|
| **Sentry** *(Recommended)* | Ubiquitous, excellent Rails integration, errors **+ tracing**; `sentry-ruby`+`sentry-rails`. |
| Honeybadger | You want errors **+ uptime + cron check-ins** in one simple product. |
| AppSignal | You want errors **+ full APM/metrics** in a single all-in-one. |
| (Rollbar) | Also fine; same shape — pick by team familiarity/pricing. |

- **Pick an error tracker — don't rely on log grepping.** It captures exceptions with
  backtrace, request context, user, and release; alerts you; and de-dupes.
- The DSN/API key is a **secret** (`../rails-security/`); set `environment` + `release`
  so errors are attributed to the right deploy.
- **Scrub PII/secrets** before sending (`before_send`/param filtering) — your
  `config.filter_parameters` (`../rails-security/`) should extend to error payloads.

## Structured logs

```ruby
# Gemfile: gem "lograge"
# config/environments/production.rb
config.lograge.enabled = true
config.lograge.formatter = Lograge::Formatters::Json.new      # one JSON line per request
config.lograge.custom_options = ->(event) { { request_id: event.payload[:request_id], host: event.payload[:host] } }
```

- Rails' default logs are multi-line and noisy; **lograge** collapses each request to
  one structured (JSON) line — far easier to ship to and query in a log aggregator.
- **Log to STDOUT** in containers (`config.logger = ActiveSupport::Logger.new(STDOUT)`
  / `RAILS_LOG_TO_STDOUT`) so the platform/Kamal collects them.
- **Tag with `request_id`** so you can trace one request across log lines.
- **Never log secrets/PII** — `filter_parameters` (`../rails-security/`) redacts
  sensitive params; verify tokens/passwords show as `[FILTERED]`.

## Metrics & APM

- **APM** (Application Performance Monitoring) — AppSignal, Datadog, New Relic, Scout,
  or Sentry tracing — shows slow endpoints, slow queries, throughput, and where time
  goes per request. Pair with `../rails-performance/` to *fix* what APM *finds*.
- **Metrics** — request rate, error rate, latency percentiles (p50/p95/p99), queue
  depth (`../rails-jobs/`), DB connection pool usage. Emit via your APM or
  Prometheus/StatsD.
- **The four golden signals** — latency, traffic, errors, saturation — are the
  baseline dashboard to put up first.

## Uptime / synthetic checks

- An external monitor (UptimeRobot, Pingdom, Honeybadger, Better Stack) hits `/up`
  (or a deeper health endpoint, [health-and-zero-downtime.md](health-and-zero-downtime.md))
  from outside and alerts when it fails — catches "the whole box is down," which
  internal metrics can't report.
- Monitor **cron/recurring jobs** with check-ins (dead-man's-switch) so a *silently
  not running* scheduler (`../rails-jobs/`) is noticed.

## Common Pitfalls

- **No error tracker** → you learn about bugs from users, late; wire one before
  launch.
- **Logging secrets/PII** → compliance + security incident; extend `filter_parameters`
  and scrub error payloads.
- **Multi-line logs to a file in a container** → hard to query, lost on redeploy; JSON
  to STDOUT + an aggregator.
- **APM with no owner** → dashboards no one reads; alert on the golden signals, route
  alerts to a human.
- **Uptime check on a too-shallow endpoint** → `/up` is 200 while the DB is down;
  decide whether you need a dependency-aware check.
- **No release tagging** → can't tell which deploy introduced an error; set
  `release`/`environment`.

## Verify

```bash
# Error tracker initializes:
bin/rails runner "p (defined?(Sentry) ? Sentry.initialized? : (defined?(Honeybadger) || defined?(Appsignal) || 'add an error tracker'))" 2>/dev/null
# Structured logging on in production config:
grep -rn "lograge" config/ 2>/dev/null && echo "✓ lograge configured" || echo "(enable structured logs)"
# Secrets are filtered from logs/errors:
bin/rails runner "p Rails.application.config.filter_parameters" | grep -iE "password|token|secret" && echo "✓ sensitive params filtered"
# Logs go to STDOUT in prod (container-friendly):
grep -rniE "log_to_stdout|STDOUT|stdout" config/environments/production.rb config/boot.rb 2>/dev/null || echo "(log to STDOUT in containers)"
```
