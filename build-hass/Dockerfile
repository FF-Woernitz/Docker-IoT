FROM homeassistant/home-assistant:stable

# Remove app from manifest
RUN sed -i.bak '/prefer_related_applications/,+3 d' /usr/src/homeassistant/homeassistant/components/frontend/__init__.py
