# Use official Tomcat image
FROM tomcat:9.0-jdk11

# Remove default apps
RUN rm -rf /usr/local/tomcat/webapps/*

# Copy WAR into container
COPY maven-wrapper.war /usr/local/tomcat/webapps/ROOT.war

# Expose port
EXPOSE 8080
