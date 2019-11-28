FROM debian:stable-slim
RUN apt-get update && apt-get install -y apache2-utils nginx openssh-server
RUN mkdir -p /run/sshd
RUN mkdir -p /etc/fargate
COPY htpasswd /etc/fargate/.htpasswd
COPY nginx-config /etc/nginx/sites-available/default
CMD ["/bin/bash","-c","/usr/bin/ssh-keygen -A && /usr/sbin/sshd && service nginx start && sleep infinity"]
