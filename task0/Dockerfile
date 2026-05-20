# Use the latest Ubuntu image as the base for this container
FROM ubuntu:latest

# Update the package index so apt knows about the latest available versions
RUN apt-get update

# Upgrade all installed packages to their latest versions
# -y auto-confirms all prompts so the build does not hang waiting for input
RUN apt-get upgrade -y

# Default command executed when the container starts
# Exec form (JSON array) runs the binary directly without a shell process
CMD ["echo", "Hello, World!"]
