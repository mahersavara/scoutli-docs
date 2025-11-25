# Project Overview: Scoutli

## Summary
**Goal**: Build a "Local Discovery Hub" (Scoutli) for sharing and discovering local "hidden gems" and community initiatives.
Scoutli is a Ruby on Rails application focused on travel discoveries. Users can share, rate, and comment on "discoveries" (locations/spots).

## Tech Stack
- **Framework**: Ruby on Rails 8.1.1
- **Language**: Ruby
- **Database**: PostgreSQL
- **Frontend**: Hotwire (Turbo + Stimulus), CSS Bundling
- **Web Server**: Puma

## Key Dependencies
- **Authentication**: `devise`
- **Authorization**: `cancancan`
- **Geocoding**: `geocoder` (Discoveries have lat/long)
- **Tagging**: `acts-as-taggable-on`
- **Database**: `pg`

## Data Models
- **User**: Managed via Devise. Has roles.
- **Discovery**: The core content. Has name, description, location (city, country, lat/long), and user association.
- **Rating**: Users can rate discoveries.
- **Comment**: Users can comment on discoveries.
- **Tag**: Discoveries can be tagged.

## Routes
- **Root**: `discoveries#index`
- **Discoveries**: CRUD operations, with nested comments and ratings.
- **Users**: Devise routes for authentication.

## Database Schema
- `users`: email, encrypted_password, role, etc.
- `discoveries`: name, description, city, country, latitude, longitude, user_id.
- `comments`: body, user_id, discovery_id.
- `ratings`: value, user_id, discovery_id.
- `tags` & `taggings`: Standard acts-as-taggable-on schema.

## Development Context (from feature_commit.md)
- **Recent Migration**: Switched from SQLite3 to PostgreSQL due to compatibility issues.
- **Frontend**: Uses Leaflet.js for maps. Had issues with `leaflet.css` (fixed via CDN) and Stimulus controller loading (fixed in `application.js`).
- **Current State**: The project is functionally complete according to the PRD. Map display and "Find Near Me" features were the last items being debugged.
