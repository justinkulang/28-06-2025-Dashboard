# Mikrotik Hotspot User Management Dashboard

This project provides a web-based dashboard for managing Mikrotik Hotspot users, including features for user creation, batch generation, profile management, and activity monitoring. It incorporates security best practices like session management, CSRF protection, and guidance for production deployment.

## Features

*   **User Management:** Create, edit, delete, and view hotspot users.
*   **Batch User Creation:** Generate multiple voucher-style users at once.
*   **Profile Management:** Manage hotspot user profiles from the Mikrotik router.
*   **Active Sessions:** View and disconnect active hotspot users.
*   **Voucher Generation:** Export user batches as printable HTML or PDF vouchers with QR codes.
*   **Analytics:** Basic analytics on data usage by profile and top users.
*   **Secure Access:**
    *   Web application login system using Flask-Login (session-based).
    *   CSRF protection for all state-changing operations using Flask-WTF.
    *   Enhanced backend input validation for API endpoints.
*   **Internationalization (i18n):** Support for multiple languages (English, Arabic, French).
*   **Configurable:** Key settings managed via `config.json`.
*   **Production Ready:** Includes Gunicorn configuration and guidance for HTTPS setup.

## Prerequisites

*   Python 3.7+
*   pip (Python package installer)
*   A Mikrotik router with the API service enabled.
*   Network connectivity between the server running this application and the Mikrotik router.
*   (Optional, for PDF export) System dependencies for WeasyPrint (see WeasyPrint documentation for your OS).

## Setup and Installation

1.  **Clone Repository:**
    ```bash
    git clone <repository_url>
    cd <repository_directory>
    ```

2.  **Create Virtual Environment (Recommended):**
    ```bash
    python -m venv venv
    source venv/bin/activate  # On Windows: venv\Scripts\activate
    ```

3.  **Install Dependencies:**
    ```bash
    pip install -r requirements.txt
    ```
    The `requirements.txt` file contains pinned versions for stable builds. You can update these or generate your own environment's specific versions using `pip freeze > requirements.txt` after testing.

4.  **Initial Configuration (`config.json`):**
    *   Upon first run, or if `config.json` is missing, a default configuration file will be created.
    *   **Web Application Admin:**
        *   A default admin user for the web dashboard is created with credentials:
            *   Username: `admin`
            *   Password: `changeme` (this is the initial password for the default hash).
        *   **IMPORTANT:** Change this default password immediately after the first login using the "Change Admin Password" feature in the "Settings" tab of the dashboard.
        *   The application uses `werkzeug.security` for password hashing. If you ever need to reset the password manually (e.g., if you're locked out and cannot use the UI), you would need to generate a new password hash and update the `password_hash` value in the `app_admin` section of `config.json`. You can use the provided `passwordg_generator.py` script for this:
            ```bash
            python passwordg_generator.py
            ```
            This script will prompt you to enter a new password and will output the corresponding hash to replace in `config.json`.
    *   **Mikrotik Connection:**
        *   Configure your Mikrotik router details (host, API username, API password, port) either by:
            1.  Manually editing `config.json` before the first run.
            2.  Using the web application's "Settings" page after logging in with the default admin credentials. The application will not be able to manage the router until these details are correctly configured.
    *   **Log File Location:** The default log file is `mikrotik_dashboard.log`. You can change this in `config.json` under `server.log_file`.
    *   **Permissions:** Ensure `config.json` has restrictive file permissions (e.g., readable only by the user running the application server) as it contains sensitive Mikrotik credentials. For example, on Linux/macOS: `chmod 600 config.json`.

## Running the Application

### Development

For development purposes, you can use the Flask development server:
```bash
python app.py
```
This server is convenient but not suitable for production. The debug mode is sourced from `config.json` (`server.debug`), which now defaults to `false`.

### Production (Recommended)

For production, it is highly recommended to use a production-grade WSGI server like Gunicorn, and to run the application behind a reverse proxy like Nginx for HTTPS termination and serving static files.

1.  **Using Gunicorn:**
    A `gunicorn_config.py` file is provided. It attempts to load server host and port from `config.json`.
    Run Gunicorn with:
    ```bash
    gunicorn --config gunicorn_config.py app:app
    ```
    Ensure Gunicorn is installed (`pip install gunicorn`).

2.  **Further Production Setup:**
    Refer to the "Production Deployment" section below for crucial details on HTTPS, environment variables, etc.

## Production Deployment

When deploying this application to a production environment, several considerations should be taken into account for security, reliability, and performance.

### `SECRET_KEY` Configuration
For session security, Flask uses a `SECRET_KEY`.
*   **Action Required:** Set the `FLASK_SECRET_KEY` environment variable to a strong, unique, and random string. Do not use the default fallback key in production.
*   The application will use the environment variable if set, otherwise, it falls back to a hardcoded development key and issues a warning.

### Debug Mode
*   **Action Required:** Ensure that `debug` is set to `false` in the `server` section of your `config.json` for production. The application now defaults this to `false` if the key is missing or a new config is generated.

### WSGI Server (Gunicorn)
*   The provided `gunicorn_config.py` sets up Gunicorn to bind to the host and port specified in `config.json` (defaulting to `0.0.0.0:5000`).
*   It also sets a recommended number of worker processes.
*   You can customize `gunicorn_config.py` further for advanced Gunicorn settings (e.g., logging, timeouts).

### HTTPS Setup (Recommended)
For production, it is strongly recommended to serve the application over HTTPS. The typical setup involves running Gunicorn locally and using a reverse proxy like Nginx or Apache in front of it to handle HTTPS termination.

**Why HTTPS is Crucial:**
*   **Security:** Encrypts data between the user's browser and the server.
*   **Data Integrity:** Ensures data is not tampered with during transit.
*   **User Trust:** Browsers mark HTTP sites as "not secure."

**Example Nginx Configuration:**
(This example assumes Gunicorn is listening on `127.0.0.1:5000`)

```nginx
server {
    listen 80;
    server_name your_domain.com; # Replace with your actual domain

    # Redirect all HTTP traffic to HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name your_domain.com; # Replace with your actual domain

    # SSL Certificate paths
    ssl_certificate /etc/letsencrypt/live/your_domain.com/fullchain.pem; # Adjust path (e.g., from Let's Encrypt)
    ssl_certificate_key /etc/letsencrypt/live/your_domain.com/privkey.pem; # Adjust path
    
    # Recommended SSL settings (consult current best practices)
    # ssl_protocols TLSv1.2 TLSv1.3;
    # ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';
    # ssl_prefer_server_ciphers off;
    # Add HSTS header (optional, but recommended)
    # add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    # (Optional) Serve static files directly with Nginx for better performance
    # location /static {
    #     alias /path/to/your/project/static; # Adjust to your app's static folder
    #     expires 7d;
    #     access_log off;
    # }

    location / {
        proxy_pass http://127.0.0.1:5000; # Must match Gunicorn's bind address
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
**Notes for Nginx:**
*   Replace `your_domain.com` with your domain.
*   Adjust SSL certificate paths. Consider using Certbot from Let's Encrypt for free certificates.
*   Ensure Gunicorn (via `gunicorn_config.py` and `config.json`) binds to `127.0.0.1:5000` if Nginx is on the same machine. If Gunicorn binds to `0.0.0.0`, ensure your firewall is configured appropriately.

### Logging
*   The application is configured to log to both the console and a file.
*   The default log file is `mikrotik_dashboard.log` (configurable in `config.json` via `server.log_file`).
*   Console and file log levels are also configurable in `config.json` (`server.log_level_console`, `server.log_level_file`).
*   When using Gunicorn, its own logging mechanisms (e.g., `accesslog`, `errorlog` in `gunicorn_config.py`) can also be used to capture stdout/stderr from the application.

### Pinned Dependencies
*   `requirements.txt` includes pinned versions for all dependencies to ensure stable and reproducible builds.
*   If you modify your environment or update packages, it's good practice to regenerate this file with your current working set: `pip freeze > requirements.txt`.

### Other Production Considerations (from previous README section)
*   **Database:** For more robust data storage than `config.json` (especially for user credentials if not using a fixed admin user), consider using a proper database system.
*   **Backups:** Implement regular backups of your application data and configurations.
*   **Monitoring:** Set up monitoring for your application and server to track performance and errors.
*   **Firewall:** Configure a firewall to only allow necessary traffic to your server (e.g., ports 80 and 443).

## Translations (i18n)

This application uses Flask-Babel for internationalization.
*   Supported languages: English (default), Arabic, French.
*   Translations are stored in the `translations` directory.
*   To add or update translations:
    1.  Extract messages: `pybabel extract -F babel.cfg -o messages.pot .`
    2.  Initialize a new language (e.g., for Spanish 'es'): `pybabel init -i messages.pot -d translations -l es`
    3.  Update existing languages: `pybabel update -i messages.pot -d translations`
    4.  Compile translations: `pybabel compile -d translations`
    (Ensure you have Babel installed and `babel.cfg` correctly configured if you modify translatable files.)

**Note on Translation Management:**
For improved long-term maintainability, especially as the number of translatable strings grows, consider the following:
*   The primary list of translatable strings for the JavaScript frontend is currently defined within `app.py` in the `/api/translations` route. While functional, a more standard approach would be to have these strings directly in the JavaScript or HTML, allowing `pybabel extract` to find them (requires proper configuration in `babel.cfg` for JavaScript extraction).
*   Alternatively, if keeping them centralized in Python, ensure all new user-facing strings added to the JavaScript frontend are also added to the `translations` dictionary in `app.py` and wrapped with `_()` there.

*(License section would go here if applicable)*
*(Contributing guidelines would go here if applicable)*

## Automated Testing (Recommended)

To ensure the long-term stability, maintainability, and reliability of this application, implementing automated tests is highly recommended. Automated tests can catch regressions early, make refactoring safer, and provide confidence when adding new features.

Consider the following types of tests:

*   **Backend Unit Tests (e.g., using `pytest`):**
    *   Test individual functions and class methods in `app.py`, especially within the `RouterOSService` to mock Mikrotik API interactions and verify logic for user/profile management, data parsing, and error handling.
    *   Test Flask route handlers for correct responses, status codes, and basic input validation logic (though much of this is now more robust).
    *   Test the `ConfigLoader` class for loading and updating configurations.

*   **Backend Integration Tests:**
    *   Test the interaction between different components, such as a route handler calling a service method that interacts with a (mocked) Mikrotik API.

*   **Frontend Unit Tests (e.g., using Jest, Vitest):**
    *   Test individual JavaScript functions in `static/js/dashboard.js` for DOM manipulation logic, data formatting, and simple UI interactions.

*   **Frontend End-to-End (E2E) Tests (e.g., using Cypress, Playwright):**
    *   Simulate user interactions in a real browser environment to test complete user flows, such as logging in, creating a user, viewing sessions, and exporting data.

**Benefits of Automated Testing:**
*   **Early Bug Detection:** Catch issues before they reach users.
*   **Safer Refactoring:** Make changes to the codebase with more confidence.
*   **Improved Code Quality:** Writing testable code often leads to better design.
*   **Living Documentation:** Tests can serve as examples of how the code is intended to be used.
