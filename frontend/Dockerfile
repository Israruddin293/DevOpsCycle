# # Multi-stage build for optimized image size
# FROM python:3.11-slim as builder

# WORKDIR /app

# # Install dependencies
# COPY requirements.txt .
# RUN pip install --no-cache-dir --user -r requirements.txt

# # Final stage
# FROM python:3.11-slim

# # Create non-root user
# RUN groupadd -r appuser && useradd -r -g appuser appuser

# WORKDIR /app

# # Copy dependencies from builder to correct location
# COPY --from=builder /root/.local /home/appuser/.local

# # Copy application code
# COPY --chown=appuser:appuser app.py .
# COPY --chown=appuser:appuser requirements.txt .
# COPY --chown=appuser:appuser templates templates/

# # Set PATH for user-installed packages
# ENV PATH=/home/appuser/.local/bin:$PATH \
#     PYTHONPATH=/app

# # Create necessary directories with proper permissions
# RUN mkdir -p /tmp /home/appuser && \
#     chown -R appuser:appuser /tmp /home/appuser /app

# # Switch to non-root user
# USER appuser

# # Expose port
# EXPOSE 8080

# # Run application
# CMD ["python", "-m", "gunicorn", "--bind", "0.0.0.0:8080", "--workers", "2", "--timeout", "120", "app:app"]



# Tested One

# Builder stage
FROM python:3.11-slim as builder

WORKDIR /app

# Install dependencies into user site
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Final stage
FROM python:3.11-slim

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Copy installed Python packages
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser . .

# Ensure user-local binaries are on PATH
ENV PATH=/home/appuser/.local/bin:$PATH

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8080

# Start the app
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "2", "--timeout", "120", "app:app"]