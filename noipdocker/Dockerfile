FROM resin/rpi-raspbian:latest

# Install cron and ddclient
RUN apt-get -y update && apt-get -y install cron nano ddclient

# Configure crontab
ADD crontab /etc/cron.d/ipupdater-cron
RUN chmod 0644 /etc/cron.d/ipupdater-cron
RUN crontab /etc/cron.d/ipupdater-cron

# Configure ddclient (ip updater)
ADD ddclient.conf /etc/ddclient.conf

# Run in foreground
CMD ["cron", "-f"]
