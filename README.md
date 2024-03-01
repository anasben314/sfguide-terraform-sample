# Add Required Provider
terraform init

# Create Snowflake Database
CREATE DATABASE TF_DEMO;

# Create Snowflake Warehouse
CREATE WAREHOUSE TF_DEMO WAREHOUSE_SIZE='SMALL' AUTO_SUSPEND=60;

# Create Snowflake Role
CREATE ROLE TF_DEMO_SVC_ROLE;

# Grant Privileges to Role on Database
GRANT USAGE ON DATABASE TF_DEMO TO ROLE TF_DEMO_SVC_ROLE;

# Create Snowflake Schema
CREATE SCHEMA TF_DEMO;

# Grant Privileges to Role on Schema
GRANT USAGE ON SCHEMA TF_DEMO TO ROLE TF_DEMO_SVC_ROLE;

# Grant Privileges to Role on Warehouse
GRANT USAGE ON WAREHOUSE TF_DEMO TO ROLE TF_DEMO_SVC_ROLE;

# Generate RSA Key Pair (Use a tool or library to generate an RSA key pair with 2048 bits)

# Create Snowflake User
CREATE USER tf_demo_user DEFAULT_WAREHOUSE = TF_DEMO DEFAULT_ROLE = TF_DEMO_SVC_ROLE DEFAULT_NAMESPACE = TF_DEMO.TF_DEMO RSA_PUBLIC_KEY = 'your_rsa_public_key_here';

# Grant MONITOR Privileges to Role on User
GRANT MONITOR ON USER tf_demo_user TO ROLE TF_DEMO_SVC_ROLE;

# Grant Role to User
GRANT ROLE TF_DEMO_SVC_ROLE TO USER tf_demo_user;
