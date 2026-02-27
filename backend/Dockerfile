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

# # Set PATH for user-installed packages
# ENV PATH=/home/appuser/.local/bin:$PATH \
#     PYTHONPATH=/app

# # Create necessary directories with proper permissions
# RUN mkdir -p /tmp /home/appuser && \
#     chown -R appuser:appuser /tmp /home/appuser /app

# # Switch to non-root user
# USER appuser

# # Expose port
# EXPOSE 5000

# # Run application
# CMD ["python", "-m", "gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "--timeout", "120", "app:app"]



# Testing Docker File

# Multi-stage build
FROM python:3.11-slim as builder

WORKDIR /app

# Install dependencies into builder
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Final stage
FROM python:3.11-slim

WORKDIR /app

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Copy installed packages from builder
COPY --from=builder /root/.local /home/appuser/.local

# Copy entire application code
COPY --chown=appuser:appuser . .

# Set PATH for user-installed packages
ENV PATH=/home/appuser/.local/bin:$PATH

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 5000

# Start app with gunicorn
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "--timeout", "120", "app:app"]