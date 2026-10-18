# Website Downloader readiness review

Static review: 18 October 2026, master at 130ad63d7163c19df64322556ca9c260eef353be. Status: retained for later use; NOT approved to run or deploy. No installation, execution, target downloads or tests were performed. This is a proposed plan, not a passing security assessment.

Upstream: https://github.com/AhmadIbrahiim/Website-downloader. Preserve attribution and unchanged LICENSE.md (MIT; Copyright (c) 2019 Ahmed Ibrhim). The tool licence does not license downloaded websites or assets.

## Workflow check

Reviewed both files under .github/workflows before editing. codeql-analysis.yml triggers on pushes to master, pull requests targeting master and a weekly schedule. It uses checkout@v2 and CodeQL init/autobuild/analyze@v1. dependabot.yml contains weekly dependency-update configuration, no on/jobs workflow; its placement inside workflows should be reviewed before relying on dependency automation.

This documentation is on review/readiness only. No master change or pull request is made, avoiding the observed push/PR triggers. No workflow is enabled, dispatched or modified. Scheduled configuration is pre-existing and separate. Modernize and review CI permissions before any later implementation PR.

## Existing protections

wget/index.js uses execFile with separate arguments rather than shell interpolation, HTTP(S) parsing, random per-job work directories, guarded cleanup paths and socket-specific progress. socket/socket.js allows one job per connection and cancels wget on disconnect. Preserve these improvements and regression-test them.

## Required hardening

- Network/SSRF: current parseTarget allows private, loopback, link-local and metadata targets, credentials and arbitrary ports. Normalize URLs; reject userinfo, non-HTTP(S), unsupported ports and every non-public IPv4/IPv6 destination, including mapped/encoded variants. Validate all DNS answers and every redirect, recursive page and asset destination. Prevent DNS rebinding with an enforcing egress proxy/firewall; first-URL checks alone are insufficient. Prefer an explicit hostname allowlist. Deny direct outbound bypass, inherited proxy settings, LAN and metadata routes. Keep TLS validation and robots restrictions; bound redirects.
- Resource limits: recursive mirror mode has no finite depth/file count. wget quota 100m is not a hard disk limit: one completed file can overshoot it. Add depth, file count, per-file/total bytes, retry and connection limits, validated configuration, actual filesystem quota, CPU/memory/process caps and one global job across all sockets.
- Lifecycle: the five-minute timer is cleared before ZIP creation, and cancellation only kills wget. Supervise the entire download/compress/delivery job with a hard deadline and complete process termination. Keep work and ZIPs in a disposable job volume. Remove partial archives on errors, enforce expiry/startup sweeping, and verify cleanup after success, failure, disconnect and restart without following symlinks outside the job root.
- Access: bin/www listens on all interfaces. Requests lack authentication; the event token is not a credential. app.js statically serves public/, while archiver/index.js writes public/sites ZIPs without expiry/access control. Bind to loopback for private single-user use; validate Host/origin and limit request sizes/rate. Authenticate HTTP, Socket.IO and per-job archive retrieval if shared access is ever introduced. Store ZIPs outside the public web root and deliver as attachments to the owning session.
- Dependencies/UI: package.json includes archiver ^3.1.1, Express ~4.19.2, Socket.IO/client ^2.5.0 and other old dependencies; no test script exists. Audit direct/transitive dependencies and lockfile against current advisories in the approved environment, upgrade to maintained compatible releases, pin supported Node LTS/GNU wget and regenerate the lockfile. Review lifecycle scripts and remove unused packages after tracing use; test server/client protocol compatibility. Remove external analytics/CDN scripts and bundle necessary UI assets. Audit success alone does not prove safety.
- Content/rights: use only owned or explicitly authorized sites/assets. Record source/date and permission for photos, fonts, code, copy and branding; retain notices and replace assets without reuse rights. Never submit credentials/cookies or bypass authentication/paywalls. Captured HTML/JS/SVG are untrusted: never auto-open or serve them from a trusted origin; inspect in a separate offline sandbox.

## Smallest isolated test proposal — approval required

Where: an ephemeral private Linux container/VM on an already-authorized test host. No public service, credentials, home-directory mounts, Docker socket, persistent integrations or spending. Non-root, read-only application files, disposable writable job volume and restricted networking. Source changes remain in GitHub. If no suitable isolated host is available, agree the environment first; do not substitute the ordinary Mac desktop.

After explicit approval only: review a pinned candidate snapshot and small new test harness; install reviewed dependencies using npm ci --ignore-scripts in a disposable build stage restricted to the package registry, review any necessary lifecycle step separately, then remove registry access. Run node --test for the new harness and supervised node ./bin/www with GNU wget against a synthetic fixture server only. No npm start/dev, existing Actions, deployment or real website is needed. Deny general Internet, LAN and metadata. URL-policy tests use mocked DNS/transport, never real private endpoints. Any fixture-only networking exception must be explicit and unavailable in normal operation.

Proposed test envelope: one job, depth 1, 20 files, 1 MiB/file, 5 MiB transfer budget, 16 MiB hard writable-volume cap, 60 seconds including ZIP, 256 MiB memory, one CPU and 32 processes. These are test targets, not current guarantees.

Exact acceptance tests:

1. Fixture HTML, CSS, image and one linked page yield a readable ZIP with rewritten links, expected hashes and no extra files.
2. Reject malformed/empty URLs, file/ftp/data, userinfo, disallowed ports, localhost, 127/8, private/link-local/metadata, IPv6 loopback/ULA/link-local and mapped IPv4. Reject mixed public/private DNS, changed DNS, forbidden redirects and cross-host recursive assets. Recording mocks/proxy prove zero forbidden connections.
3. Shell metacharacters remain one argument; no extra process/file side effect occurs.
4. Endless links, oversized response, many files, stalls and redirect loops stop within limits. Hard disk cap and total deadline apply during ZIP creation too.
5. Foreign-origin/unauthenticated socket and ZIP access fail; only loopback listens; a second socket cannot run concurrently; another session cannot obtain the archive.
6. Missing wget, 403, robots exclusion, empty result, nonzero exit with partial files, full/unwritable volume and ZIP errors produce explicit failure/partial status, never false success or server crash.
7. Disconnect/timeout during download AND compression leave no child, temporary file or partial ZIP. Delivery, expiry and restart sweep leftovers. A sentinel outside the job root survives every cleanup case.
8. Record pinned runtime/wget, lockfile/audit review and client compatibility. Assert no external UI analytics/CDN requests and no execution of captured scripts by the harness.

Before merge, record candidate commit, environment/image digest, commands, dependency findings, denied-network evidence, ZIP hashes and pass/fail output in GitHub. Failed network, access, bound or cleanup tests block use.

## Limitations and release gate

This captures publicly delivered pages/assets, not server source, databases, authenticated business logic or a faithful working copy of every SPA. JavaScript-rendered pages and third-party assets may be incomplete. Treat it as an authorized import/reference aid followed by a reviewed rebuild.

Release requires reviewed hardening on a separate branch, specific isolated-execution approval, passing evidence and an agreed private operating procedure. No public hosting, automatic live-site download or production use is approved.
