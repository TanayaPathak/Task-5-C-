# Task-5-C-


#include <iostream>
#include <vector>
#include <string>
#include <sqlite3.h>
#include <ctime>

class Book {
public:
    std::string title;
    std::string author;
    std::string isbn;
    bool available;

    Book(std::string title, std::string author, std::string isbn, bool available = true) {
        this->title = title;
        this->author = author;
        this->isbn = isbn;
        this->available = available;
    }
};

class Borrower {
public:
    std::string name;
    std::string contact;
    std::vector<Book*> borrowed_books;

    Borrower(std::string name, std::string contact) {
        this->name = name;
        this->contact = contact;
    }
};

class Transaction {
public:
    Book* book;
    Borrower* borrower;
    std::string due_date;
    bool returned;

    Transaction(Book* book, Borrower* borrower, std::string due_date) {
        this->book = book;
        this->borrower = borrower;
        this->due_date = due_date;
        this->returned = false;
    }
};

class Library {
private:
    sqlite3* conn;
    sqlite3_stmt* stmt;

public:
    Library() {
        sqlite3_open("library.db", &conn);
        create_tables();
    }

    ~Library() {
        sqlite3_close(conn);
    }

    void create_tables() {
        sqlite3_exec(conn, "CREATE TABLE IF NOT EXISTS books (id INTEGER PRIMARY KEY AUTOINCREMENT, title TEXT, author TEXT, isbn TEXT, available INTEGER)", 0, 0, 0);
        sqlite3_exec(conn, "CREATE TABLE IF NOT EXISTS borrowers (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, contact TEXT)", 0, 0, 0);
        sqlite3_exec(conn, "CREATE TABLE IF NOT EXISTS transactions (id INTEGER PRIMARY KEY AUTOINCREMENT, book_id INTEGER, borrower_id INTEGER, due_date DATE, returned INTEGER, FOREIGN KEY (book_id) REFERENCES books(id), FOREIGN KEY (borrower_id) REFERENCES borrowers(id))", 0, 0, 0);
    }

    void add_book(Book* book) {
        sqlite3_prepare_v2(conn, "INSERT INTO books (title, author, isbn, available) VALUES (?, ?, ?, ?)", -1, &stmt, 0);
        sqlite3_bind_text(stmt, 1, book->title.c_str(), -1, SQLITE_STATIC);
        sqlite3_bind_text(stmt, 2, book->author.c_str(), -1, SQLITE_STATIC);
        sqlite3_bind_text(stmt, 3, book->isbn.c_str(), -1, SQLITE_STATIC);
        sqlite3_bind_int(stmt, 4, book->available);
        sqlite3_step(stmt);
        sqlite3_finalize(stmt);
    }

    std::vector<Book*> search_books(std::string keyword) {
        std::vector<Book*> books;
        sqlite3_prepare_v2(conn, "SELECT * FROM books WHERE title LIKE ? OR author LIKE ? OR isbn LIKE ?", -1, &stmt, 0);
        sqlite3_bind_text(stmt, 1, ("%" + keyword + "%").c_str(), -1, SQLITE_STATIC);
        sqlite3_bind_text(stmt, 2, ("%" + keyword + "%").c_str(), -1, SQLITE_STATIC);
        sqlite3_bind_text(stmt, 3, ("%" + keyword + "%").c_str(), -1, SQLITE_STATIC);
        while (sqlite3_step(stmt) == SQLITE_ROW) {
            std::string title = std::string(reinterpret_cast<const char*>(sqlite3_column_text(stmt, 1)));
            std::string author = std::string(reinterpret_cast<const char*>(sqlite3_column_text(stmt, 2)));
            std::string isbn = std::string(reinterpret_cast<const char*>(sqlite3_column_text(stmt, 3)));
            bool available = sqlite3_column_int(stmt, 4);
            books.push_back(new Book(title, author, isbn, available));
        }
        sqlite3_finalize(stmt);
        return books;
    }

    void check_out_book(int book_id, int borrower_id, std::string due_date) {
        sqlite3_prepare_v2(conn, "UPDATE books SET available = 0 WHERE id = ?", -1, &stmt, 0);
        sqlite3_bind_int(stmt, 1, book_id);
        sqlite3_step(stmt);
        sqlite3_finalize(stmt);

        sqlite3_prepare_v2(conn, "INSERT INTO transactions (book_id, borrower_id, due_date, returned) VALUES (?, ?, ?, 0)", -1, &stmt, 0);
        sqlite3_bind_int(stmt, 1, book_id);
        sqlite3_bind_int(stmt, 2, borrower_id);
        sqlite3_bind_text(stmt, 3, due_date.c_str(), -1, SQLITE_STATIC);
        sqlite3_step(stmt);
        sqlite3_finalize(stmt);
    }

    void return_book(int transaction_id) {
        sqlite3_prepare_v2(conn, "UPDATE transactions SET returned = 1 WHERE id = ?", -1, &stmt, 0);
        sqlite3_bind_int(stmt, 1, transaction_id);
        sqlite3_step(stmt);
        sqlite3_finalize(stmt);

        sqlite3_prepare_v2(conn, "SELECT * FROM transactions WHERE id = ?", -1, &stmt, 0);
        sqlite3_bind_int(stmt, 1, transaction_id);
        sqlite3_step(stmt);
        int book_id = sqlite3_column_int(stmt, 1);
        sqlite3_finalize(stmt);

        sqlite3_prepare_v2(conn, "UPDATE books SET available = 1 WHERE id = ?", -1, &stmt, 0);
        sqlite3_bind_int(stmt, 1, book_id);
        sqlite3_step(stmt);
        sqlite3_finalize(stmt);
    }

    double calculate_fine(int transaction_id) {
        sqlite3_prepare_v2(conn, "SELECT * FROM transactions WHERE id = ?", -1, &stmt, 0);
        sqlite3_bind_int(stmt, 1, transaction_id);
        sqlite3_step(stmt);
        std::string due_date = std::string(reinterpret_cast<const char*>(sqlite3_column_text(stmt, 3)));
        sqlite3_finalize(stmt);

        std::tm tm = {};
        strptime(due_date.c_str(), "%Y-%m-%d", &tm);
        std::time_t due_time = std::mktime(&tm);
        std::time_t current_time = std::time(nullptr);
        double fine = std::max(0.0, difftime(current_time, due_time) / (60 * 60 * 24) * 0.5);
        return fine;
    }
};

int main() {
    Library library;

    while (true) {
        std::cout << "1. Add Book" << std::endl;
        std::cout << "2. Search Books" << std::endl;
        std::cout << "3. Check Out Book" << std::endl;
        std::cout << "4. Return Book" << std::endl;
        std::cout << "5. Calculate Fine" << std::endl;
        std::cout << "6. Exit" << std::endl;

        std::string choice;
        std::cout << "Enter your choice (1-6): ";
        std::cin >> choice;

        if (choice == "1") {
            std::string title, author, isbn;
            std::cout << "Enter book title: ";
            std::cin.ignore();
            std::getline(std::cin, title);
            std::cout << "Enter book author: ";
            std::getline(std::cin, author);
            std::cout << "Enter book ISBN: ";
            std::getline(std::cin, isbn);
            Book* book = new Book(title, author, isbn);
            library.add_book(book);
            std::cout << "Book added successfully." << std::endl;
        }
        else if (choice == "2") {
            std::string keyword;
            std::cout << "Enter search keyword: ";
            std::cin.ignore();
            std::getline(std::cin, keyword);
            std::vector<Book*> books = library.search_books(keyword);
            if (!books.empty()) {
                for (Book* book : books) {
                    std::cout << "Title: " << book->title << ", Author: " << book->author << ", ISBN: " << book->isbn << ", Available: " << (book->available ? "Yes" : "No") << std::endl;
                }
            }
            else {
                std::cout << "No books found." << std::endl;
            }
        }
        else if (choice == "3") {
            int book_id, borrower_id;
            std::string borrower_name, borrower_contact, due_date;
            std::cout << "Enter book ID to check out: ";
            std::cin >> book_id;
            std::cout << "Enter borrower name: ";
            std::cin.ignore();
            std::getline(std::cin, borrower_name);
            std::cout << "Enter borrower contact: ";
            std::getline(std::cin, borrower_contact);
            std::cout << "Enter due date (YYYY-MM-DD): ";
            std::getline(std::cin, due_date);
            Borrower* borrower = new Borrower(borrower_name, borrower_contact);
            library.check_out_book(book_id, borrower_id, due_date);
            std::cout << "Book checked out successfully." << std::endl;
        }
        else if (choice == "4") {
            int transaction_id;
            std::cout << "Enter transaction ID to return book: ";
            std::cin >> transaction_id;
            library.return_book(transaction_id);
            std::cout << "Book returned successfully." << std::endl;
        }
        else if (choice == "5") {
            int transaction_id;
            std::cout << "Enter transaction ID to calculate fine: ";
            std::cin >> transaction_id;
            double fine = library.calculate_fine(transaction_id);
            std::cout << "Fine for this transaction: $" << fine << std::endl;
        }
        else if (choice == "6") {
            std::cout << "Exiting..." << std::endl;
            break;
        }
        else {
            std::cout << "Invalid choice. Please try again." << std::endl;
        }
    }

    return 0;
}
