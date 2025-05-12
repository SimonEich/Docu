# SQL

## Python SQLite
Creating a DB, and creating tables.

    import sqlite3

    def create_connection(db_name="example.db"):
    
    """Create and return a connection to the SQLite database."""
    conn = sqlite3.connect(db_name)
    return conn

    def create_tables(conn):
    """Create two connected tables: users and posts."""
    cursor = conn.cursor()

    Enable foreign key constraint
    cursor.execute("PRAGMA foreign_keys = ON")

    # Create users table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            email TEXT UNIQUE NOT NULL
        )
    """)

    # Create posts table with foreign key to users
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS posts (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            title TEXT NOT NULL,
            content TEXT,
            FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
        )
    """)

    conn.commit()
    print("Tables created successfully.")

## Python: Edit Data 
### Delete Data
        self.cursor.execute("DELETE FROM words WHERE word = ?", (word,))
        self.conn.commit()
### Edit Data
        self.cursor.execute(
            "UPDATE words SET word = ?, translation = ? WHERE word = ?",
            (new_word, new_translation, old_word)
        )
        self.conn.commit()
### Read Data
        self.cursor.execute("SELECT * FROM words")
        data_db = self.cursor.fetchall()
### Random Data
        self.cursor.execute("SELECT * FROM words WHERE count = 0 ORDER BY RANDOM() LIMIT 1;")
### Insert Data
        self.cursor.execute("INSERT INTO words VALUES (?, ?, ?)", (word, translation, 0))        
