---
title: "Real - Time Chat App using Python Sockets and Tkinter"
layout: post
---

Hey there,

Today, we're going to make a Python chat application, which is a digital chatroom, where people can send messages to each other. This chat app consists of two parts: a server (server.py) and a client (client.py). These parts work together to enable communication over the internet.

We'll break down how these parts interact and function. Below is the code for server.py:

{% highlight ruby %}
import socket
import threading

class ChatServer:
    def __init__(self):
        self.clients_list = []
        self.last_received_message = ""
        self.create_listening_server()

    def create_listening_server(self):
        local_ip, local_port = '127.0.0.1', 10319
        self.server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.server_socket.bind((local_ip, local_port))
        self.server_socket.listen(5)
        print("Listening for incoming messages...")
        self.receive_messages_in_thread()

    def receive_messages(self, so):
        while True:
            incoming_buffer = so.recv(256)
            if not incoming_buffer:
                break
            self.last_received_message = incoming_buffer.decode('utf-8')
            self.broadcast_to_all_clients(so)
        so.close()

    def broadcast_to_all_clients(self, senders_socket):
        for client, _ in self.clients_list:
            if client is not senders_socket:
                client.sendall(self.last_received_message.encode('utf-8'))

    def receive_messages_in_thread(self):
        while True:
            client, (ip, port) = self.server_socket.accept()
            self.clients_list.append((client, (ip, port)))
            print(f'Connected to {ip}:{port}')
            threading.Thread(target=self.receive_messages, args=(client,)).start()

if __name__ == "__main__":
    ChatServer()
{% endhighlight %}

Explanation:
1. Imagine the server as a hub where people gather to chat. It's like a club that opens at a specific location (127.0.0.1) and a particular time (port 10319).
2. The server creates a way for people (clients) to connect to it. It's like setting up a front door and welcoming anyone who wants to come in.
3. Inside this club (server), there's a host (ChatServer) who manages everything. This host has a list to keep track of all the people (clients) who are currently in the club. It also remembers the last message received.
4. When a person (client) enters the club, the host starts listening to what that person has to say. It's like the host's job is to listen to each person's conversation.
5. If someone in the club says something, the host makes sure to share that message with everyone else. It's like if someone talks, the host makes sure everyone hears it.
6. The host also organizes things efficiently. When a new person enters the club, the host introduces them to everyone and starts listening to what they have to say by putting their conversation on a different "thread."

Below is the code for client.py:
{% highlight ruby %}
import tkinter as tk
from tkinter import Frame, Scrollbar, Label, Entry, Text, Button, messagebox
import socket
import threading

class GUI:
    def __init__(self, master):
        self.root = master
        self.chat_transcript_area = None
        self.name_widget = None
        self.enter_text_widget = None
        self.client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.initialize_socket()
        self.initialize_gui()
        self.listen_for_incoming_messages_in_a_thread()

    def initialize_socket(self):
        self.client_socket.connect(('127.0.0.1', 10319))

    def initialize_gui(self):
        self.root.title("Socket Chat")
        self.root.resizable(0, 0)
        self.display_name_section()
        self.display_chat_entry_box()
        self.display_chat_box()

    def listen_for_incoming_messages_in_a_thread(self):
        thread = threading.Thread(target=self.receive_message_from_server)
        thread.start()

    def receive_message_from_server(self):
        while True:
            buffer = self.client_socket.recv(256)
            if not buffer:
                break
            message = buffer.decode('utf-8')
            if "joined" in message:
                user = message.split(":")[1]
                message = user + " has joined"
                self.chat_transcript_area.insert('end', message + '\n')
                self.chat_transcript_area.yview('end')
            else:
                self.chat_transcript_area.insert('end', message + '\n')
                self.chat_transcript_area.yview('end')

    def display_name_section(self):
        frame = tk.Frame()
        label = Label(frame, text='Enter Your Name Here!', font=("arial", 13, "bold"))
        label.pack(side='left', pady=20)
        self.name_widget = Entry(frame, width=60, font=("arial", 13))
        self.name_widget.pack(side='left', anchor='e', pady=15)
        self.join_button = Button(frame, text="Join", width=10, command=self.on_join)
        self.join_button.pack(side='right', padx=5, pady=15)
        frame.pack(side='top', anchor='nw')

    def display_chat_box(self):
        frame = tk.Frame()
        label = Label(frame, text='Chat Box', font=("arial", 12, "bold"))
        label.pack(side='top', padx=270)
        self.chat_transcript_area = Text(frame, width=60, height=10, font=("arial", 12))
        scrollbar = Scrollbar(frame, command=self.chat_transcript_area.yview, orient='vertical')
        self.chat_transcript_area.config(yscrollcommand=scrollbar.set)
        self.chat_transcript_area.bind('<KeyPress>', lambda e: 'break')
        self.chat_transcript_area.pack(side='left', padx=15, pady=10)
        scrollbar.pack(side='right', fill='y', padx=1)
        frame.pack(side='left')

    def display_chat_entry_box(self):
        frame = tk.Frame()
        label = Label(frame, text='Enter Your Message Here!', font=("arial", 12, "bold"))
        label.pack(side='top', anchor='w', padx=120)
        self.enter_text_widget = Text(frame, width=50, height=10, font=("arial", 12))
        self.enter_text_widget.pack(side='left', pady=10, padx=10)
        self.enter_text_widget.bind('<Return>', self.on_enter_key_pressed)
        frame.pack(side='left')

    def on_join(self):
        name = self.name_widget.get().strip()
        if name:
            self.name_widget.config(state='disabled')
            self.client_socket.send(f"joined:{name}".encode('utf-8'))

    def on_enter_key_pressed(self, event):
        if len(self.name_widget.get()) == 0:
            messagebox.showerror("Enter your name", "Enter your name to send a message")
        else:
            self.send_chat()
            self.clear_text()

    def clear_text(self):
        self.enter_text_widget.delete(1.0, 'end')

    def send_chat(self):
        sender_name = self.name_widget.get().strip() + ": "
        data = self.enter_text_widget.get(1.0, 'end').strip()
        message = (sender_name + data).encode('utf-8')
        self.chat_transcript_area.insert('end', message.decode('utf-8') + '\n')
        self.chat_transcript_area.yview('end')
        self.client_socket.send(message)
        self.enter_text_widget.delete(1.0, 'end')

    def confirm_quit(self):
        if messagebox.askokcancel("Quit", "Do you want to quit?"):
            self.client_socket.close()
            self.root.destroy()

if __name__ == '__main__':
    root = tk.Tk()
    gui = GUI(root)
    root.protocol("WM_DELETE_WINDOW", gui.confirm_quit)
    root.mainloop()
{% endhighlight %}

Explanation:
1. The client is like someone who wants to join the chat club. To do that, they need to know where the club is (127.0.0.1) and what time it opens (port 10319).
2. The client also needs a special way to interact with the club, which is like having a phone to call the club.
3. The client has a chat window with their name, a place to type messages, and a button to join the club. It's like they have a chatroom with their name on it.
4. The client is a good listener too. It keeps an ear out for messages from the club all the time.
5. When a message comes in from the club, the client reads it and shows it in their chat window. If someone new joins the club, the client notices and lets everyone know.
6. To join the club, the client types their name and clicks the "Join" button. It's like they introduce themselves to the club.
7. When they want to send a message, they type it and hit "Enter" or click the send button. The message shows up in the chat window, and the club gets it too.
8. If the client decides to leave the club, there's a polite way to say goodbye. It's like asking if they're sure they want to leave before they go.

In short, the server is like a chat club's host, and the client is like someone attending the club. They communicate over the internet, and the code helps them chat and coordinate effectively.

To run the python, type this to cmd.
<img width="298" alt="server" src="https://github.com/lazypotatogamer/lazypotatogamer.github.io/assets/91038494/127a6ea2-6a0f-454d-8267-fd82b5ca09b1">

<img width="339" alt="client" src="https://github.com/lazypotatogamer/lazypotatogamer.github.io/assets/91038494/e20e489d-c8f0-46cf-a594-01f1cb03029f">

when the program starts to run, it will show this:
<img width="815" alt="output" src="https://github.com/lazypotatogamer/lazypotatogamer.github.io/assets/91038494/e29f13a4-7235-4f3d-9b8e-951f5bf29383">

you can now chat with other user/client in the chat room.
<img width="815" alt="output1" src="https://github.com/lazypotatogamer/lazypotatogamer.github.io/assets/91038494/a9789a56-a17f-4a59-9ea2-20690db1acc9">









