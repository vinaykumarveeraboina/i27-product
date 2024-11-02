# Official OPENJDK18 image as the base image
FROM openjdk:18

# Create the directory /opt/i27/ , inwhich we store our application.
RUN mkdir -p /opt/i27/

# Setting the working directory to /opt/i27
WORKDIR /opt/i27/

# Define a build-time variable called JAR_SOURCE to specify the location of the JAR file
ARG JAR_SOURCE

# Copy the JAR File 
COPY ["${JAR_SOURCE}", "/opt/i27/i27car-product.jar"]

# Grant RWX persmission on this directory 
RUN chmod 777 /opt/i27/

# Expose port 8132 to allow external access to application.
EXPOSE 8132

# Specify the command to run the Spring Based application using the JAVA JAR File
CMD ["java", "-jar", "/opt/i27/i27car-product.jar"]

# CMD ["java", "-jar", "/*.jar"]