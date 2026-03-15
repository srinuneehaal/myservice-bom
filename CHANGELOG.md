# Changelog

All notable changes to this project will be documented in this file.

The project follows semantic versioning.

## [Unreleased]

### Changed

- raised the BOM Java baseline from 17 to 21
- changed GitHub Actions publishing to run on `main` pushes instead of release-only events

## [1.0.0-SNAPSHOT] - 2026-03-14

### Added

- initial `myservice-parent` BOM project for Spring Boot microservices
- centralized version properties for Spring Boot, `org.json`, and `commons-io`
- GitHub Actions CI workflow for BOM validation
- GitHub Packages publishing configuration for `srinuneehaal/myservice-bom`
