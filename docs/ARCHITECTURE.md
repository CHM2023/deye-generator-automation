# Architecture

This note describes how Deye Generator Automation is put together. It contains no code, no control thresholds and no timing constants other than the five-minute control cadence. The source is private; see the README for contact details.

## Components

- **Web application.** A Laravel application behind a login. It serves the dashboard, accepts operator commands (automation on and off, parameters, the six-slot schedule, grid mode on and off, per-inverter force, release and charge commands), hosts the admin portal for accounts, and registers browsers for push notifications. It never writes to an inverter without first taking the device lock.
- **Cycle runner.** A console command that performs exactly one control cycle and exits. Cron launches it every five minutes. A daemon variant loops in-process for Docker deployments and traps termination signals so it can restore the inverters before it dies.
- **Settings check.** A separate console command, also launched by cron, that compares every inverter with the saved start level and schedule, writes back anything that drifted, and leaves a report for the dashboard.
- **Round rules.** Pure decision functions with no I/O. Given an observation (the clock, the local time of day and the latest sample per inverter) and the round state, they return one decision: do nothing, start a round, join an inverter, shed an inverter, begin the end sequence, run a restore pass, or finish. Because they are pure they are tested against a simulated plant.
- **Supervisor.** The runner around the rules. It fetches telemetry, keeps the last good samples, lets the rules decide, executes the decision through the actuator, writes the state file before and after every change, and appends one row per cycle to the cycle log.
- **Actuator stack.** Three layers between the supervisor and the cloud. The control layer applies configuration guards: the serial must be one of the configured inverters, a start level must lie inside the writable window, and a register write must pass a fingerprint check on the neighbouring registers before anything is sent. The commander submits orders, polls them to completion, honours the dry-run switch and applies the re-send policy. The API client handles the token, retries reads and sends writes exactly once.
- **Grid mode.** A module with its own state file that runs once per cycle after the round step, under the same device lock. It tracks grid presence, backs up and holds the schedules while the grid is present, and restores them when it goes.
- **Stores.** Plain files on disk for everything that must survive a process: the round state, the automation switches and parameters, the grid state, the snapshots taken before a manual force, the device baseline (what each inverter last reported for its charge current and generator cap), the last seen generator and grid states for notifications, the last settings-check report, and the cycle log.
- **Notifier and alerter.** The notifier sends Web Push messages to every registered browser. The alerter always logs and can also deliver to a Telegram bot or a webhook.
- **Device lock.** A cache-backed lock shared by the cycle runner, the settings check and every web request that writes. One writer at a time, across processes.
- **Deye Cloud.** The only path to the inverters. Telemetry is read from it, and every command is an order queued on it and executed by the inverter's logger.

## The control loop

One cycle runs in this order.

1. Take the device lock, or skip this tick if another process holds it. A skipped tick is normal and the next one carries on.
2. Reload the operator's settings from disk and rebuild the configuration, so a parameter changed on the dashboard applies to this cycle.
3. Load the round state file. Under cron the age of the file matters: a recent file is this scheduler's own round and is continued; an old file means the scheduler stopped, and the inverters are restored rather than the round continued. A dry-run round found by a live process is discarded, and a live round found by a dry-run process is refused, because a dry-run process could only pretend to restore.
4. Write a heartbeat carrying the cadence this runner actually runs at, so the dashboard can judge whether the automation is alive.
5. Fetch the latest telemetry for all inverters in one call. Records without a timestamp or a battery level are dropped; a failed fetch keeps the previous samples, and their age decides whether they are still usable.
6. Update the per-sample counters once per new cloud sample: consecutive samples with the generator off, consecutive samples with the main unit not drawing. The same sample is never counted twice.
7. Prune the pending queue: inverters that rose above the join level are dropped, inverters that had no fresh data at the start of the round and report now are appended.
8. Decide. If the grid mode holds the schedules, no round starts and a running round ends. In the idle phase a round starts only on a natural generator start by an inverter at its start level. In the round phase the end conditions are checked first, then the shed condition, then the next join. In the ending and fault phases the restore pass runs until every touched inverter is confirmed back.
9. Execute. A join reads the inverter's settings and schedule, records the snapshot and the intent on disk, then writes the schedule targets and the start level, each verified by read-back. When a join succeeds the rules are asked again in the same cycle, so every inverter that fits joins the same generator run. A restore pass switches generator charging off on every touched inverter, waits for the generator to be seen off, then puts the start level back, the schedule targets back and generator charging on again, in that order. A finish requires a final read-back of every touched inverter.
10. Save the state file and append the cycle row.
11. Run the grid mode step with the same observation.
12. Compare the generator and grid states with the last seen ones and send a push notification for every change.
13. Release the lock.

## Deye Cloud API calls, by category and in order

1. **Authentication.** A token request with hashed account credentials. The token is cached with its lifetime and refreshed before it expires. A refusal on a token that is still young is treated as a permission problem rather than expiry, with a cooldown before the next refresh attempt.
2. **Telemetry.** One "latest data" request for all configured serials at the start of every cycle. It returns battery level, battery power and voltage, load, generator power and voltage, grid power and voltage, solar power and the collection time of each inverter.
3. **Settings reads.** A custom-control order carrying a Modbus read frame for the inverter's generator-charge block (start level, charge current, enable flag, generator-input cap, grid-charge start level). The order is queued, polled until it reaches a terminal status, and the reply frame is parsed and CRC-checked. Reads are not gated by the dry-run switch and may be re-sent when their outcome is unknown, because a duplicate read cannot change the inverter.
4. **Schedule reads.** A dynamic-control read order followed by its result request, returning the six time-of-use slots with their start times, targets, power and the generator, grid and sell flags.
5. **Writes.** Only during a round, a grid hold or an operator command, and each one preceded by a read and followed by a read-back: a time-of-use update that always carries all six slots; a custom-control order carrying a Modbus write frame for a single start-level register, after the fingerprint check; and a battery charge-mode order that switches generator charging on or off. Writes are sent exactly once at the transport level. A rejected order may be resubmitted; an order with an unknown fate is never resubmitted and is reported as uncertain.
6. **Order status.** The order record is polled after every write, and orders whose fate was unknown are polled again on later cycles until they settle.

## State kept between rounds

- **Round state:** the phase (idle, round, ending, fault), the dry-run flag of the process that started the round, the round id, the start time, the main unit and a snapshot of its schedule, the pending queue in join order, the generator-input cap read from each inverter, and one record per touched inverter holding its schedule snapshot, the slots that were raised, the start level before the round, the time it joined, why it was skipped if it was, and the flags that record what was written and what was restored. Also the end reason, the fault reason, the restore and verification failure counters, the per-sample counters and the time the last round ended.
- **Automation settings:** the automation switch and the mode it was switched on in, the grid-mode switch, the three operator parameters and the six saved schedule targets.
- **Grid state:** the phase (idle, waiting, holding, restoring), the consecutive-sample counters for grid present and absent, the last counted sample time, and for each inverter the backed-up targets and grid-charge start level plus the written and restored flags and any pending order ids.
- **Manual-force snapshots:** what an inverter looked like before an operator forced it from its card, written before anything is sent and never overwritten by a second force, so release can put it back.
- **Device baseline:** the charge current and generator-input cap each inverter last reported, adopted from verified reads so the headroom estimate follows the plant rather than a number written down on the day the registers were verified.
- **Plant events:** the last seen generator and grid states, so a notification is sent only on a change.
- **Settings-check report:** what the last check found and corrected, shown on the dashboard for a day.
- **Cycle log:** one CSV row per cycle with the decision, its reason, the generator state and load, and every inverter's sample.
- **Cache:** the API token, a short-lived telemetry cache for the dashboard, the heartbeat and the device lock.

## Notification and dashboard flow

The dashboard polls two endpoints: the plant state (telemetry for every inverter, the generator load against its limits, grid presence, each card's last-read settings and schedule) and the automation state (switches, parameters, the round on file, the grid mode, the heartbeat and the settings-check notice). Telemetry is fetched from the cloud at most every few seconds whatever the page asks, and the page shows the age of the data it is looking at. Two views share the same data: the full view for developers with the inverter cards, the command log and the configuration line, and a plain-language view for operators and administrators.

Every operator write takes the device lock, so it can never overlap with a cycle. Writes to all inverters are made one inverter at a time so the page can show each one's progress, and an inverter that a round is currently handling, or whose schedule the grid mode holds, is skipped rather than written.

Push notifications are sent from the cycle, after the plant work is done, when the generator or the grid changes state. They are judged on fresh samples exactly as the automation judges them, so a notification never disagrees with the dashboard. Browsers register through the dashboard; a subscription that the browser has dropped is forgotten on the next failed delivery. Round events (started, ended, aborted, restored, settings corrected, state file not writable) go to the log and to the optional Telegram or webhook channel.

## Deployment

Production runs on shared Linux hosting where long-running processes are not allowed but cron is. The web application serves the dashboard; two cron entries run the cycle every five minutes and the settings check on its own schedule. State files live under the application's storage directory and never travel between machines, because a state file describes forced inverters on one specific installation. Accounts live in MySQL.

A Docker Compose variant exists for a VPS: a web container with nginx and php-fpm, a dedicated supervisor container running the daemon as its own process so the container runtime can signal it directly and give it time to restore before stopping, Redis for the cache and the cross-process lock, and MySQL. The two variants share the same code and state files.

Every environment starts in dry-run mode, in which every write is logged but nothing is sent, and the dashboard says so in its header. Going live is a deliberate change of one setting after a few rehearsed cycles.
