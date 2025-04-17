# ATM Simulation with CRUD Operations (Procedural Python)

```
atm_simulation/
│
├── database/
│   ├── __init__.py
│   └── db_operations.py
│
├── models/
│   ├── __init__.py
│   └── user.py
│
├── utils/
│   ├── __init__.py
│   ├── auth.py
│   └── account_operations.py
│
├── main.py
└── requirements.txt
```

## Updated Code Implementation

### 1. Database Operations (db_operations.py)

```python
import sqlite3
from passlib.hash import pbkdf2_sha256

def initialize_database():
    """Initialize the database and create tables if they don't exist"""
    conn = sqlite3.connect('database/atm.db')
    cursor = conn.cursor()
    
    # Create users table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT UNIQUE NOT NULL,
            password_hash TEXT NOT NULL,
            full_name TEXT NOT NULL,
            pin TEXT NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    
    # Create accounts table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS accounts (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            account_number TEXT UNIQUE NOT NULL,
            account_type TEXT NOT NULL,
            balance REAL DEFAULT 0.0,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY(user_id) REFERENCES users(id)
        )
    ''')
    
    # Create transactions table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS transactions (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            account_id INTEGER NOT NULL,
            transaction_type TEXT NOT NULL,
            amount REAL NOT NULL,
            description TEXT,
            timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY(account_id) REFERENCES accounts(id)
        )
    ''')
    
    conn.commit()
    conn.close()

def get_db_connection():
    """Get a database connection"""
    return sqlite3.connect('database/atm.db')

def hash_password(password):
    """Hash a password for storing"""
    return pbkdf2_sha256.hash(password)

def verify_password(stored_hash, provided_password):
    """Verify a stored password against one provided by user"""
    return pbkdf2_sha256.verify(provided_password, stored_hash)

def generate_account_number():
    """Generate a random 10-digit account number"""
    import random
    return ''.join([str(random.randint(0, 9)) for _ in range(10)])
```

### 2. User Model (user.py)

```python
import sqlite3

def create_user(username, password, full_name, pin):
    """Create a new user in the database"""
    from database.db_operations import get_db_connection, hash_password, generate_account_number
    
    password_hash = hash_password(password)
    pin_hash = hash_password(pin)
    
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Create user
        cursor.execute('''
            INSERT INTO users (username, password_hash, full_name, pin)
            VALUES (?, ?, ?, ?)
        ''', (username, password_hash, full_name, pin_hash))
        
        user_id = cursor.lastrowid
        
        # Create a default checking account
        account_number = generate_account_number()
        cursor.execute('''
            INSERT INTO accounts (user_id, account_number, account_type)
            VALUES (?, ?, ?)
        ''', (user_id, account_number, 'checking'))
        
        conn.commit()
        return True
    except sqlite3.IntegrityError:
        # Username or account number already exists
        return False
    finally:
        conn.close()

def get_user_by_username(username):
    """Retrieve a user by username"""
    from database.db_operations import get_db_connection
    
    conn = get_db_connection()
    cursor = conn.cursor()
    
    cursor.execute('SELECT * FROM users WHERE username = ?', (username,))
    user = cursor.fetchone()
    
    conn.close()
    return user

def get_user_by_id(user_id):
    """Retrieve a user by ID"""
    from database.db_operations import get_db_connection
    
    conn = get_db_connection()
    cursor = conn.cursor()
    
    cursor.execute('SELECT * FROM users WHERE id = ?', (user_id,))
    user = cursor.fetchone()
    
    conn.close()
    return user

def verify_pin(user_id, pin):
    """Verify user's PIN"""
    from database.db_operations import get_db_connection, verify_password
    
    conn = get_db_connection()
    cursor = conn.cursor()
    
    cursor.execute('SELECT pin FROM users WHERE id = ?', (user_id,))
    pin_hash = cursor.fetchone()[0]
    
    conn.close()
    return verify_password(pin_hash, pin)
```

### 3. Authentication Utilities (auth.py)

```python
from database.db_operations import verify_password

def authenticate_user(username, password):
    """Authenticate a user with username and password"""
    from models.user import get_user_by_username
    
    user = get_user_by_username(username)
    if user and verify_password(user[2], password):  # user[2] is password_hash
        return user
    return None

def login():
    """Handle user login"""
    print("\n--- ATM Login ---")
    username = input("Username: ")
    password = input("Password: ")
    
    user = authenticate_user(username, password)
    if user:
        # Verify PIN
        pin = input("Enter your 4-digit PIN: ")
        from models.user import verify_pin
        if verify_pin(user[0], pin):  # user[0] is id
            print(f"\nWelcome, {user[3]}!")  # user[3] is full_name
            return user[0]  # Return user ID
        else:
            print("\nInvalid PIN")
    else:
        print("\nInvalid username or password")
    return None

def register():
    """Handle user registration"""
    print("\n--- ATM Registration ---")
    username = input("Choose a username: ")
    password = input("Choose a password: ")
    full_name = input("Full name: ")
    pin = input("Create a 4-digit PIN: ")
    
    # Validate PIN
    if len(pin) != 4 or not pin.isdigit():
        print("\nPIN must be 4 digits")
        return
    
    from models.user import create_user
    if create_user(username, password, full_name, pin):
        print("\nRegistration successful! Account created. You can now login.")
    else:
        print("\nUsername already exists. Please choose another.")
```

### 4. Account Operations (account_operations.py)

```python
def get_user_accounts(user_id):
    """Get all accounts for a user"""
    from database.db_operations import get_db_connection
    conn = get_db_connection()
    cursor = conn.cursor()
    
    cursor.execute('''
        SELECT id, account_number, account_type, balance 
        FROM accounts 
        WHERE user_id = ?
    ''', (user_id,))
    
    accounts = cursor.fetchall()
    conn.close()
    return accounts

def display_accounts(user_id):
    """Display all accounts for a user"""
    accounts = get_user_accounts(user_id)
    
    if not accounts:
        print("\nNo accounts found.")
    else:
        print("\n--- Your Accounts ---")
        for account in accounts:
            print(f"\nAccount Number: {account[1]}")
            print(f"Type: {account[2].capitalize()}")
            print(f"Balance: ${account[3]:.2f}")
    
    return accounts

def create_account(user_id):
    """Create a new account for the user"""
    print("\n--- Create New Account ---")
    account_type = input("Account type (checking/savings): ").lower()
    
    if account_type not in ['checking', 'savings']:
        print("\nInvalid account type. Must be 'checking' or 'savings'")
        return
    
    from database.db_operations import get_db_connection, generate_account_number
    conn = get_db_connection()
    cursor = conn.cursor()
    
    account_number = generate_account_number()
    
    try:
        cursor.execute('''
            INSERT INTO accounts (user_id, account_number, account_type)
            VALUES (?, ?, ?)
        ''', (user_id, account_number, account_type))
        
        conn.commit()
        print(f"\nAccount created successfully! Account Number: {account_number}")
    except sqlite3.Error as e:
        print(f"\nError creating account: {e}")
    finally:
        conn.close()

def deposit(user_id):
    """Deposit money into an account"""
    accounts = display_accounts(user_id)
    if not accounts:
        return
    
    account_id = input("\nEnter account ID to deposit into: ")
    amount = float(input("Enter amount to deposit: "))
    
    if amount <= 0:
        print("\nAmount must be positive")
        return
    
    from database.db_operations import get_db_connection
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Update balance
        cursor.execute('''
            UPDATE accounts 
            SET balance = balance + ? 
            WHERE id = ? AND user_id = ?
        ''', (amount, account_id, user_id))
        
        if cursor.rowcount == 0:
            print("\nAccount not found or doesn't belong to you")
            return
        
        # Record transaction
        cursor.execute('''
            INSERT INTO transactions (account_id, transaction_type, amount, description)
            VALUES (?, ?, ?, ?)
        ''', (account_id, 'deposit', amount, 'ATM deposit'))
        
        conn.commit()
        print("\nDeposit successful!")
    except sqlite3.Error as e:
        print(f"\nError during deposit: {e}")
    finally:
        conn.close()

def withdraw(user_id):
    """Withdraw money from an account"""
    accounts = display_accounts(user_id)
    if not accounts:
        return
    
    account_id = input("\nEnter account ID to withdraw from: ")
    amount = float(input("Enter amount to withdraw: "))
    
    if amount <= 0:
        print("\nAmount must be positive")
        return
    
    from database.db_operations import get_db_connection
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Check balance
        cursor.execute('''
            SELECT balance FROM accounts 
            WHERE id = ? AND user_id = ?
        ''', (account_id, user_id))
        
        result = cursor.fetchone()
        if not result:
            print("\nAccount not found or doesn't belong to you")
            return
        
        balance = result[0]
        if balance < amount:
            print("\nInsufficient funds")
            return
        
        # Update balance
        cursor.execute('''
            UPDATE accounts 
            SET balance = balance - ? 
            WHERE id = ? AND user_id = ?
        ''', (amount, account_id, user_id))
        
        # Record transaction
        cursor.execute('''
            INSERT INTO transactions (account_id, transaction_type, amount, description)
            VALUES (?, ?, ?, ?)
        ''', (account_id, 'withdrawal', amount, 'ATM withdrawal'))
        
        conn.commit()
        print("\nWithdrawal successful!")
    except sqlite3.Error as e:
        print(f"\nError during withdrawal: {e}")
    finally:
        conn.close()

def transfer(user_id):
    """Transfer money between accounts"""
    accounts = display_accounts(user_id)
    if len(accounts) < 2:
        print("\nYou need at least 2 accounts to transfer")
        return
    
    from_account = input("\nEnter account ID to transfer FROM: ")
    to_account = input("Enter account ID to transfer TO: ")
    amount = float(input("Enter amount to transfer: "))
    
    if amount <= 0:
        print("\nAmount must be positive")
        return
    
    from database.db_operations import get_db_connection
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Verify accounts belong to user
        cursor.execute('''
            SELECT id FROM accounts 
            WHERE id IN (?, ?) AND user_id = ?
        ''', (from_account, to_account, user_id))
        
        if len(cursor.fetchall()) != 2:
            print("\nOne or both accounts not found or don't belong to you")
            return
        
        # Check balance
        cursor.execute('SELECT balance FROM accounts WHERE id = ?', (from_account,))
        balance = cursor.fetchone()[0]
        if balance < amount:
            print("\nInsufficient funds")
            return
        
        # Perform transfer
        # Deduct from source account
        cursor.execute('''
            UPDATE accounts 
            SET balance = balance - ? 
            WHERE id = ?
        ''', (amount, from_account))
        
        # Add to destination account
        cursor.execute('''
            UPDATE accounts 
            SET balance = balance + ? 
            WHERE id = ?
        ''', (amount, to_account))
        
        # Record transactions
        cursor.execute('''
            INSERT INTO transactions (account_id, transaction_type, amount, description)
            VALUES (?, ?, ?, ?)
        ''', (from_account, 'transfer_out', amount, f'Transfer to account {to_account}'))
        
        cursor.execute('''
            INSERT INTO transactions (account_id, transaction_type, amount, description)
            VALUES (?, ?, ?, ?)
        ''', (to_account, 'transfer_in', amount, f'Transfer from account {from_account}'))
        
        conn.commit()
        print("\nTransfer successful!")
    except sqlite3.Error as e:
        conn.rollback()
        print(f"\nError during transfer: {e}")
    finally:
        conn.close()

def view_transactions(user_id):
    """View transaction history for an account"""
    accounts = display_accounts(user_id)
    if not accounts:
        return
    
    account_id = input("\nEnter account ID to view transactions: ")
    
    from database.db_operations import get_db_connection
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Verify account belongs to user
    cursor.execute('''
        SELECT id FROM accounts 
        WHERE id = ? AND user_id = ?
    ''', (account_id, user_id))
    
    if not cursor.fetchone():
        print("\nAccount not found or doesn't belong to you")
        conn.close()
        return
    
    # Get transactions
    cursor.execute('''
        SELECT transaction_type, amount, description, timestamp 
        FROM transactions 
        WHERE account_id = ?
        ORDER BY timestamp DESC
        LIMIT 10
    ''', (account_id,))
    
    transactions = cursor.fetchall()
    
    if not transactions:
        print("\nNo transactions found for this account")
    else:
        print("\n--- Recent Transactions ---")
        for trans in transactions:
            print(f"\nType: {trans[0].capitalize()}")
            print(f"Amount: ${trans[1]:.2f}")
            print(f"Description: {trans[2]}")
            print(f"Date: {trans[3]}")
    
    conn.close()
```

### 5. Main Application (main.py)

```python
from database.db_operations import initialize_database
from utils.auth import login, register
from utils.account_operations import (
    display_accounts, create_account, 
    deposit, withdraw, transfer, view_transactions
)

def main_menu():
    """Display the main menu"""
    print("\n=== ATM Main Menu ===")
    print("1. Login")
    print("2. Register")
    print("3. Exit")
    return input("Choose an option: ")

def atm_menu(username):
    """Display the ATM menu after login"""
    print(f"\n=== ATM Menu ({username}) ===")
    print("1. View Accounts")
    print("2. Create New Account")
    print("3. Deposit")
    print("4. Withdraw")
    print("5. Transfer")
    print("6. View Transactions")
    print("7. Logout")
    return input("Choose an option: ")

def main():
    """Main application loop"""
    initialize_database()
    
    current_user = None
    current_username = None
    
    while True:
        if not current_user:
            # Not logged in - show main menu
            choice = main_menu()
            
            if choice == '1':
                current_user = login()
                if current_user:
                    current_username = get_user_by_id(current_user)[3]  # Full name
            elif choice == '2':
                register()
            elif choice == '3':
                print("\nThank you for using our ATM. Goodbye!")
                break
            else:
                print("\nInvalid choice. Please try again.")
        else:
            # Logged in - show ATM menu
            choice = atm_menu(current_username)
            
            if choice == '1':
                display_accounts(current_user)
            elif choice == '2':
                create_account(current_user)
            elif choice == '3':
                deposit(current_user)
            elif choice == '4':
                withdraw(current_user)
            elif choice == '5':
                transfer(current_user)
            elif choice == '6':
                view_transactions(current_user)
            elif choice == '7':
                current_user = None
                current_username = None
                print("\nLogged out successfully!")
            else:
                print("\nInvalid choice. Please try again.")

if __name__ == "__main__":
    main()
```

## Key Features of the ATM Simulation

1. **User Authentication**:
   - Secure registration with username, password, and 4-digit PIN
   - Login with username, password, and PIN verification

2. **Account Management**:
   - Automatic creation of checking account on registration
   - Ability to create additional accounts (checking/savings)
   - View all accounts with balances

3. **Financial Operations**:
   - Deposit money into accounts
   - Withdraw money from accounts (with balance checking)
   - Transfer between accounts
   - View transaction history

4. **Security**:
   - Password and PIN hashing
   - Proper account ownership verification
   - Transaction recording for audit purposes

5. **Database Structure**:
   - Users table with authentication info
   - Accounts table linked to users
   - Transactions table for record-keeping

## How to Use the ATM Simulation

1. Run the application:
   ```bash
   python main.py
   ```

2. Register a new user account or login with existing credentials.

3. After login, you can:
   - View your accounts and balances
   - Create new accounts
   - Deposit money
   - Withdraw money
   - Transfer between accounts
   - View transaction history
   - Logout
