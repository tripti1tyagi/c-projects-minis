# c-projects-minis
      /*  #include <iostream>
        #include <string>
        #include <vector>
        #include <thread>
        #include <mutex>
        #include <chrono>
        #include <fstream>
        #include <sstream>
        #include <iomanip>
        #include <algorithm>  // Include algorithm for std::remove

        using namespace std;

        // Mutex to handle concurrent console access
        mutex mtx;

        // Struct to store user information
        struct Message {
            string username;
            string text;
            string timestamp;  // Timestamp for each message
        };

        // Global vector to store the list of users in the chat
        vector<string> users;

        // Function to get the current timestamp
        string getCurrentTimestamp() {
            auto now = chrono::system_clock::to_time_t(chrono::system_clock::now());
            tm *ltm = localtime(&now);
            stringstream ss;
            ss << put_time(ltm, "%Y-%m-%d %H:%M:%S");
            return ss.str();
        }

        // Function to handle user input (sending messages)
        void userInput(vector<Message>& chatHistory, const string& username) {
            string message;

            // Add user to the chat list
            mtx.lock();
            users.push_back(username);
            mtx.unlock();

            while (true) {
                cout << username << ": ";
                getline(cin, message);

                // Handle commands
                if (message == "/exit") {
                    // Remove user from chat list and exit
                    mtx.lock();
                    // Erase the user from the list using the erase-remove idiom
                    users.erase(remove(users.begin(), users.end(), username), users.end());
                    mtx.unlock();
                    break;
                } 
                else if (message == "/clear") {
                    mtx.lock();
                    chatHistory.clear(); // Clear the chat history
                    mtx.unlock();
                    cout << "Chat history cleared." << endl;
                }
                else if (message == "/list") {
                    mtx.lock();
                    cout << "Users in chat: ";
                    for (const auto& user : users) {
                        cout << user << " ";
                    }
                    cout << endl;
                    mtx.unlock();
                }
                else {
                    mtx.lock();
                    // Store the message with the current timestamp
                    chatHistory.push_back({username, message, getCurrentTimestamp()});
                    mtx.unlock();
                }
            }
        }

        // Function to display the chat history
        void chatDisplay(const vector<Message>& chatHistory) {
            while (true) {
                mtx.lock();
                cout << "\n--- Chat History ---\n";

                // Display all the chat messages with timestamps
                for (const auto& msg : chatHistory) {
                    cout << "[" << msg.timestamp << "] " << msg.username << ": " << msg.text << endl;
                }

                cout << "\n--------------------\n";
                mtx.unlock();

                this_thread::sleep_for(chrono::seconds(1)); // Delay for smooth display
            }
        }

        // Function to save chat history to a file
        void saveChatHistoryToFile(const vector<Message>& chatHistory) {
            ofstream outFile("chat_history.txt");

            if (outFile.is_open()) {
                for (const auto& msg : chatHistory) {
                    outFile << "[" << msg.timestamp << "] " << msg.username << ": " << msg.text << endl;
                }
                outFile.close();
                cout << "Chat history saved to chat_history.txt" << endl;
            } else {
                cout << "Failed to save chat history." << endl;
            }
        }

        int main() {
            string username;
            vector<Message> chatHistory;  // Store chat messages

            // Ask user for their username
            cout << "Enter your username: ";
            getline(cin, username);

            // Create threads for user input and chat display
            thread inputThread(userInput, ref(chatHistory), username);  // Input thread for sending messages
            thread displayThread(chatDisplay, ref(chatHistory));  // Display thread for showing chat history

            // Join the input thread (waiting for it to finish)
            inputThread.join();  

            // Detach the display thread (this will continue indefinitely)
            displayThread.detach();  

            // Save chat history to a file when exiting
            saveChatHistoryToFile(chatHistory);

            return 0;
        */
#include <iostream>
#include <string>
#include <vector>
#include <thread>
#include <mutex>
#include <chrono>
#include <fstream>
#include <sstream>
#include <iomanip>
#include <algorithm>

using namespace std;

// Mutex to handle concurrent console access
mutex mtx;

// Condition variable for graceful shutdown of chatDisplay thread
bool chatRunning = true;

// Struct to store user information
struct Message {
    string username;
    string text;
    string timestamp;  // Timestamp for each message
};

// Global vector to store the list of users in the chat
vector<string> users;

// Function to get the current timestamp
string getCurrentTimestamp() {
    auto now = chrono::system_clock::to_time_t(chrono::system_clock::now());
    tm* ltm = localtime(&now);
    stringstream ss;
    ss << put_time(ltm, "%Y-%m-%d %H:%M:%S");
    return ss.str();
}

// Function to handle user input (sending messages)
void userInput(vector<Message>& chatHistory, const string& username) {
    string message;
    {
        // Add user to the chat list
        lock_guard<mutex> lock(mtx);
        users.push_back(username);
    }
    while (true) {
        cout << username << ": ";
        getline(cin, message);
        // Handle commands
        if (message == "/exit") {
            {
                // Remove user from chat list and exit
                lock_guard<mutex> lock(mtx);
                users.erase(remove(users.begin(), users.end(), username), users.end());
            }
            break;
        } 
        else if (message == "/clear") {
            lock_guard<mutex> lock(mtx);
            chatHistory.clear(); // Clear the chat history
            cout << "Chat history cleared." << endl;
        }
        else if (message == "/list") {
            lock_guard<mutex> lock(mtx);
            cout << "Users in chat: ";
            for (const auto& user : users) {
                cout << user << " ";
            }
            cout << endl;
        }
        else {
            lock_guard<mutex> lock(mtx);
            // Store the message with the current timestamp
            chatHistory.push_back({username, message, getCurrentTimestamp()});
        }
    }
}

// Function to display the chat history
void chatDisplay(const vector<Message>& chatHistory) 
{
    while (chatRunning) {  // Only run while chat is active
        {
            lock_guard<mutex> lock(mtx);
            cout << "\n--- Chat History ---\n";
            // Display all the chat messages with timestamps
            for (const auto& msg : chatHistory) {
                cout << "[" << msg.timestamp << "] " << msg.username << ": " << msg.text << endl;
            }
            cout << "\n--------------------\n";
        }
        this_thread::sleep_for(chrono::seconds(1)); // Delay for smooth display
    }
}

// Function to save chat history to a file
void saveChatHistoryToFile(const vector<Message>& chatHistory) 
{
    ofstream outFile("chat_history.txt");
    if (outFile.is_open()) 
    {
        for (const auto& msg : chatHistory) 
        {
            outFile << "[" << msg.timestamp << "] " << msg.username << ": " << msg.text << endl;
        }
        outFile.close();
        cout << "Chat history saved to chat_history.txt" << endl;
    } 
    else 
    {
        cout << "Failed to save chat history." << endl;
    }
}

int main() {
    vector<Message> chatHistory; 
    thread user1([](vector<Message>& chatHistory) 
    {
        string username = "Alice";
        userInput(chatHistory, username);
    }, ref(chatHistory));
    thread user2([](vector<Message>& chatHistory) 
    {
        string username = "Bob";
        userInput(chatHistory, username);
    }, ref(chatHistory));
    thread displayThread(chatDisplay, ref(chatHistory));  // Display thread for showing chat history
    // Join threads to wait for their completion
    user1.join();
    user2.join();
    chatRunning = false;  // Stop the chatDisplay thread after users finish
    displayThread.join(); // Wait for the display thread to exit
    // Save chat history to a file when exiting
    saveChatHistoryToFile(chatHistory);
    return 0;
}
