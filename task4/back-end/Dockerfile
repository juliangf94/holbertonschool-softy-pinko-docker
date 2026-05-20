FROM ubuntu:latest
# Update the package index so apt knows about the latest available versions
RUN apt-get update
# Upgrade all installed packages to their latest versions
# -y auto-confirms all prompts so the build does not hang waiting for input
RUN apt-get upgrade -y
# 
RUN apt-get install -y python3 python3-pip
# 
RUN rm /usr/lib/python*/EXTERNALLY-MANAGED
# 
RUN pip3 install Flask
# Install flask-cors to allow cross-origin requests from the frontend container
RUN pip3 install flask-cors

WORKDIR /app
# 
COPY ./api.py /app/api.py
# 
CMD ["python3", "api.py"]
