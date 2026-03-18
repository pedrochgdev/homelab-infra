# monitor

## Summary

`monitor` is the infrastructure monitoring VM hosted on `virt`. It runs the current observability stack for the homelab and provides visibility into system health and service behavior.

It is already operational and serves as the central monitoring point for the environment.

## Role

- monitoring VM
- observability node
- dashboard and metrics platform

## Operating System

- Platform: Ubuntu Server
- Version: `24.04`

## Host Platform

- Hosted on: `virt`
- Virtualization platform: Proxmox VE `9.1.0`

## Resources

- CPU: 1 socket, 1 core
- Memory: 2 GB RAM
- Disk: 25 GB
- IP: `192.168.1.50`

## Service Stack

Current monitoring services:

- Prometheus
- Grafana

## Purpose

The purpose of `monitor` is to provide:

- infrastructure visibility
- service monitoring
- dashboard-based operational awareness
- a foundation for improving observability maturity over time

## Current State

`monitor` is active and functioning.

It currently supports the environment at a practical level, but its presentation and operational maturity are still expected to improve.

## Current Improvement Areas

The main current improvement areas are:

- make Grafana dashboards more professional
- improve organization and presentation of monitoring data
- expand observability maturity as the homelab grows

## Notes

This VM is intentionally separate from application hosts so monitoring remains centralized and not tied to a single service node.