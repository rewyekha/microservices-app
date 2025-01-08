---
description: Server.py app demo
icon: snake
---

# Auth service

***

### Auth-Service

This `Auth-Service` is a simple Flask-based microservice that provides two main functionalities: **user authentication** and **JWT validation**. It connects to a PostgreSQL database to verify user credentials and generates a JSON Web Token (JWT) upon successful login. Below is a detailed explanation of the components and their purpose.

***

#### **Key Functionalities**

1.  **Database Connection**\
    The `get_db_connection` function establishes a connection to a PostgreSQL database using environment variables to retrieve sensitive credentials like the host, database name, username, and password.

    ```python
    def get_db_connection():
        conn = psycopg2.connect(host=os.getenv('DATABASE_HOST'),
                                database=os.getenv('DATABASE_NAME'),
                                user=os.getenv('DATABASE_USER'),
                                password=os.getenv('DATABASE_PASSWORD'),
                                port=5432)
        return conn
    ```
2.  **Login Endpoint (`/login`)**

    * **Route**: `/login`
    * **Method**: `POST`
    * This endpoint validates user credentials against the PostgreSQL database.
    * **Process**:
      * Accepts basic authentication credentials (`username` and `password`) through the request headers.
      * Queries the database (using the `AUTH_TABLE` environment variable to determine the table name) to find a matching user.
      * If valid credentials are provided, a JWT is generated using the `CreateJWT` function and returned to the user.
      * If authentication fails, a `401 Unauthorized` response is sent back.

    ```python
    @server.route('/login', methods=['POST'])
    def login():
        # Fetch credentials from request
        auth = request.authorization
        ...
        # Verify user credentials from the database
        query = f"SELECT email, password FROM {auth_table_name} WHERE email = %s"
        res = cur.execute(query, (auth.username,))
        ...
        # Generate JWT if valid
        return CreateJWT(auth.username, os.environ['JWT_SECRET'], True)
    ```
3.  **JWT Creation (`CreateJWT`)**\
    This function generates a JWT using the `jwt` library. The token contains the following claims:

    * `username`: The username of the authenticated user.
    * `exp`: Token expiration date (set to 1 day from the current UTC time).
    * `iat`: Issued-at timestamp.
    * `admin`: A boolean indicating whether the user has admin privileges.

    ```python
    def CreateJWT(username, secret, authz):
        return jwt.encode(
            {
                "username": username,
                "exp": datetime.datetime.now(tz=datetime.timezone.utc) + datetime.timedelta(days=1),
                "iat": datetime.datetime.now(tz=datetime.timezone.utc),
                "admin": authz,
            },
            secret,
            algorithm="HS256",
        )
    ```
4.  **JWT Validation Endpoint (`/validate`)**

    * **Route**: `/validate`
    * **Method**: `POST`
    * This endpoint verifies the validity of a JWT sent in the `Authorization` header of the request.
    * **Process**:
      * Extracts the token from the header.
      * Decodes the token using the secret key specified in the `JWT_SECRET` environment variable.
      * If the token is invalid or expired, a `401 Unauthorized` response is returned.
      * If valid, the decoded token is returned as the response.

    ```python
    @server.route('/validate', methods=['POST'])
    def validate():
        encoded_jwt = request.headers['Authorization']
        ...
        decoded_jwt = jwt.decode(encoded_jwt, os.environ['JWT_SECRET'], algorithms=["HS256"])
        return decoded_jwt, 200
    ```

***

#### **Environment Variables**

The application relies on several environment variables for configuration:

* **`DATABASE_HOST`**: The hostname of the PostgreSQL database.
* **`DATABASE_NAME`**: The name of the PostgreSQL database.
* **`DATABASE_USER`**: The username for the database connection.
* **`DATABASE_PASSWORD`**: The password for the database connection.
* **`AUTH_TABLE`**: The name of the table containing user authentication data.
* **`JWT_SECRET`**: The secret key used to encode and decode JWTs.

***

#### **How It Works**

1. A client sends a `POST` request to the `/login` endpoint with credentials in the authorization header.
2. The service verifies the credentials against the PostgreSQL database.
3. If the credentials are valid, a JWT is returned to the client.
4. The client can use this JWT to access protected resources by including it in the `Authorization` header.
5. To validate the JWT, the client sends it to the `/validate` endpoint, which decodes the token and verifies its authenticity.

***

#### **Usage Instructions**

1. Set up the required environment variables.
2. Start the server using `python <filename>.py`.
3. Use tools like Postman or `curl` to interact with the `/login` and `/validate` endpoints.

***

This service provides a basic foundation for user authentication and can be extended with additional security measures, such as password hashing, rate-limiting, or two-factor authentication.
