import socket
import threading
import paramiko
import logging

# 1. Setup Logging - This captures attacker activity to a file
logging.basicConfig(
    filename="honeypot_activity.log",
    level=logging.INFO,
    format="%(asctime)s - %(message)s"
)

# 2. Define the SSH Server Interface
class SSHServer(paramiko.ServerInterface):
    def _init_(self):
        self.event = threading.Event()

    def check_auth_password(self, username, password):
        # Log the attempted credentials
        logging.info(f"Login attempt - Username: {username} | Password: {password}")
        # Always return Denied to keep them trying different passwords
        return paramiko.AUTH_FAILED

    def get_allowed_auths(self, username):
        return "password"

# 3. Handle Incoming Connections
def handle_connection(client_socket):
    try:
        # Generate or load a host key (ssh-keygen -t rsa -f server.key)
        host_key = paramiko.RSAKey.generate(2048)
        transport = paramiko.Transport(client_socket)
        transport.add_server_key(host_key)
        
        server = SSHServer()
        transport.start_server(server=server)
        
    except Exception as e:
        print(f"Error: {e}")
    finally:
        transport.close()

# 4. Main Listener
def start_honeypot(port=2222):
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server_socket.bind(("0.0.0.0", port))
    server_socket.listen(100)

    print(f"[*] Honeypot active on port {port}...")
    
    while True:
        client_socket, addr = server_socket.accept()
        print(f"[*] Connection detected from {addr[0]}")
        client_thread = threading.Thread(target=handle_connection, args=(client_socket,))
        client_thread.start()

if _name_ == "_main_":
    start_honeypot()
