# Changelog

## [3.2.1] - 2026-11-22

### Fixed

- Keep the Debian package systemd-only by removing the OpenRC `/etc/init.d` and runit service scripts, preventing Debian/Ubuntu from passing an OpenRC script to `update-rc.d`.
- Publish the OpenRC service script as a separate download for OpenRC users.

### Changed

- Bump the Debian package version to 3.2.1.

## [3.2.0] - 2026-11-20

### Added

- Track learned process habits by executable name and a hash of the command line, reducing collisions between distinct invocations.
- Support `thresholds.cpu_count_mode` values `physical` and `logical` for CPU percentage normalization.
- Write cgroup setup and fallback information to `/var/lib/process-sentry/status.json`.
- Include the systemd service definition.

### Changed

- Replace the daemon and service definitions with the supplied v3 updates.
- Bump the Debian package version to 3.2.0.

## [3.1.1] - 2026-07-05

- Adjust habit decay defaults and correct GameMode daemon exclusions.

## [3.1.0] - 2026-04-11

- Add gaming and Appify handling, cgroup memory limits, and performance and reliability fixes.
