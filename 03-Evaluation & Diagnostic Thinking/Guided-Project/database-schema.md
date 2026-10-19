# TaskBridge — Database Schema

## 1. Overview

This document describes the proposed database structure for TaskBridge, an AI-powered application that transforms unclear requests into structured and actionable plans.

The database is designed to store user accounts, submitted requests, clarification details, and generated action plans when these features are implemented.

**Important:** This is a proposed schema. It must be checked against the actual application and existing Supabase database before any SQL is executed.

## 2. Database Technology

- Database: PostgreSQL
- Backend platform: Supabase
- Authentication: Supabase Auth
- Data protection: Row Level Security (RLS)

## 3. Proposed Tables

The proposed schema contains the following tables:

1. `profiles` — optional user profile information.
2. `requests` — requests submitted by users.
3. `clarification_questions` — questions created to clarify incomplete requests.
4. `action_plans` — structured plans generated from requests.
5. `action_plan_tasks` — individual tasks belonging to an action plan.

The final implementation may use fewer tables if the application does not require all these features.

## 4. Table Definitions

### 4.1 Profiles

Stores optional profile information associated with an authenticated user.

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key; references `auth.users.id` |
| `display_name` | TEXT | Optional display name |
| `created
