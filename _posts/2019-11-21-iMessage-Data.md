---
title: iMessage Data
---

The first of my data that I decided to look into was my iMessage data. This data was conveniently stored in a database file, which I found in `/Users/$USER/Library/Messages/chat.db`. Despite spending years working with data, I don't have much experience working with _databases_. Usually data comes to me as a group of plain text of .wav files, or sometimes stored in a csv file. So examining the iMessage data has been a learning experience for me. Throughout, I've found the tutorials at [SQLite Tutorial](www.sqlitetutorial.net) extremely helpful.

What's in the database?
------------------
First off, I had to find out what kind of database it was, which I was able to do using the `file` command:

```
$ file chat.db
chat.db:     SQLite 3.x database
```

Great! I already had sqlite installed on my computer, but if you don't you can find it at [this link](https://www.sqlite.org/index.html) or if you're on a mac, I recommend using [Homebrew](https://brew.sh/). (`brew install sqlite3`).

I loaded the database and used the `.tables` command to see what tables were in there.

```
$ sqlite3 chat.db
SQLite version 3.8.10.2 2015-05-20 18:17:19
Enter ".help" for usage hints.
sqlite> .tables
_SqliteDatabaseProperties  deleted_messages
attachment                 handle
chat                       message
chat_handle_join           message_attachment_join
chat_message_join
```

There are two handy ways to look at what the different columns of a table are: `.schema <table name>` or `pragma table_info('<table name>')`. 'Schema' provides a lot of detail about the types of columns, indexes, and triggers in the table, but for someone just trying to get a handle on what's going on here, I found 'pragma table\_info' much more helpful. As an example, here is the table info for the chat table.


```
sqlite> pragma table_info('chat');
cid     name    type    notnull dflt_value      pk
0       ROWID   INTEGER 0               1
1       guid    TEXT    1               0
2       style   INTEGER 0               0
3       state   INTEGER 0               0
4       account_id      TEXT    0               0
5       properties      BLOB    0               0
6       chat_identifier TEXT    0               0
7       service_name    TEXT    0               0
8       room_name       TEXT    0               0
9       account_login   TEXT    0               0
10      is_archived     INTEGER 0       0       0
11      last_addressed_handle   TEXT    0               0
12      display_name    TEXT    0               0
13      group_id        TEXT    0               0
14      is_filtered     INTEGER 0       0       0
15      successful_query        INTEGER 0       1       0
```

Now I know the columns of the table, but it's still hard to understand what information is contained in them. For that I found it most useful to `SELECT` a sample of different rows to look at. For example, `select distinct room_name, service_name from chat limit 10;` shows me 10 examples of unique room\_name/service\_name combinations:

```
sqlite> select distinct room_name, service_name from chat limit 10;
room_name service_name
     Jabber
     SMS
     iMessage
chat102606977248217513  iMessage
chat112595321236621046  SMS
chat115662523429788197  iMessage
chat119033988564590150  iMessage
chat121498848935427410  iMessage
chat124227341444912508  iMessage
chat125704926518609154  SMS
```


Summary
=======
After much poking around, here is a summary of what's going on in the iMessage chat.db.

| Table                | Description  |
| ------- | ---------- |
| message | Table with a row for each message ever sent, including the text for the message, the handle_id for the chat was with, the date, whether it 'is\_from\_me`, etc |
| handle   |  Table matching handle_ids (as given in the message table), with the id of the handle (email, phone number, etc). Also includes country and service (ex. SMS). |
| chat | Table grouping messages into chats. This is basically what enables group chats. Each chat has an id, and messages and handles are mapped to that id (see below). |
| chat\_message\_join | Table associating chats with messages. |
| chat\_handle\_join | Table associating chats with handles. Each chat can have multiple handles, and each handle can be in multiple chats. |
| attachment | Table with all attachments, including their files names and transfer status. |
| message\_attachment\_join | Table associating messages with attachment info |
| deleted\_messages | This is empty for me so it's hard to tell what would be in it. Presumably deleted messages. |


Answering questions
-----------------
Now that I have a grasp on the database, I want to use it to answer some questions.

### Who sends me the most messages?

To answer this question, I need to count ids. I could count handle\_ids, but each actual id (email, phone number, etc) may correspond to multiple handle\_ids. So instead I do an inner join on the handle table to convert to id. In addition, the handle\_id often encodes the person the message was associated with, including both messages to and from the person. So if I want to know who sends me the most messages, I also to filter on whether the messages is\_from\_me.


```
SELECT id, count(id) /* show ids and the number of times they occur */
FROM message /* primary table to query is the message table */
INNER JOIN handle on handle.ROWID = message.handle_id /* combine message with handle table */
WHERE is_from_me = 0 /* count only messages from other people */
GROUP BY id /* group unique ids together for counting */
ORDER BY count(id) DESC /* see results in order of most to least count */
LIMIT 10; /* only show 10 results */
```

This gives me a satisfying list of ids and message counts:
```
id      count(id)
<husband's phone number>    23937
<friend A email>    18554
<friend B phone number>    10873
<friend A phone number>    4567
<friend C email>     4084
<husband's email> 3353
<in-law A phone number>    3009
<friend B icloud>     2353
<in-law B phone number>    2145
<school project partner>      1768
```

A few interesting observations:
* People have sent me a lot of messages! Especially my husband!
* There are actually only 7 individuals on this list. I haven't found a good way to group people by true identity (a problem I see reflected in my chat program as well), so when I communicate with the same person using multiple methods, they show up twice. I could solve this by creating a table mapping id to true name, but that's a project for a different day.
* The in-law group chat is busy! But maybe unsurprisingly although these are way up on my "people who message me" list, they're much lower on the "people I message" list (their messages are sent to a group chat and typically not directed so much at me.)
* Almost all the people on this list have had at least 5 years (usually more) to build up this impressive message count. One exception is the school project partner, who I mainly messaged with over the course of one year. It was a big project!

### Who have I talked to most in the last week
(This refers to the last week of the dataset, which I "froze" for this project a few weeks ago)

For this query I have to introduce the date field. Timestamp are given in [Mac absolute time](https://www.epochconverter.com/coredata). One way to do this is to stay in Mac absolute time. One week is 604,800 seconds, so if we subtract that from the maximum date, we get the messages in the past week.

```
SELECT id, count(id)
FROM message
INNER JOIN handle on handle.ROWID = message.handle_id
WHERE is_from_me = 0
AND date > (SELECT MAX(date) FROM message) - 604800  /* only show results in the last week */
GROUP BY id
ORDER BY count(id) DESC
LIMIT 10;
```

Another option is to convert the Mac Absolute Time date to datetime, and then use datetime to subtract 7 days. Because this requires converting between Mac Absolute Time and Unix Epoch twice, finding the max, and doing a subtraction, it's pretty difficult to read.

```
SELECT id, count(id)
FROM message
INNER JOIN handle on handle.ROWID = message.handle_id
WHERE is_from_me = 0
AND datetime(date + strftime('%s','2001-01-01'), 'unixepoch') >  (select max(datetime(datetime(date + strftime('%s','2001-01-01'), 'unixepoch'), '-7 day')) from message)  /* only show results in the last week */
GROUP BY id
ORDER BY count(id) DESC
LIMIT 10;
```

Either way, this is the result:


```
id      count(id)
<husband's email> 58
<friend B phone number>    36
<friend D phone number>    11
<in-law A phone number>    5
<my phone number>    3
<husband's phone number>    2
<in-law B phone number>    2
<doctor phone number>    1
<friend E phone number>    1
<friend F phone number>    1
```

* A much smaller number of messages. In fact, you don't have to message me much at all to make the list
* More new friends made the list, since they're not out-weighed by people who have been messaging me for 10 years. 
* Much more heavily skewed towards phone numbers, as I've moved away from "Jabber" (gmail) and towards true iMessage.
* In-law group chat still going strong!
* Why is my own phone number on the list? I messaged myself a few photos and links as the easiest way to transfer them from my phone to my computer

On the other hand, if I look at the top people who *received* messages from me, there are only 4 from that week (including myself).


### What are my most common messages?

Just liked we counted and grouped by id above, now we have to count and group my message text. All the information is in the message table, which makes for a simpler query.


```
SELECT text, count(text)
FROM message
GROUP BY text
ORDER BY count(text) DESC
LIMIT 10;
```

I'm also going to throw in a `where is_from_me = 0` and `= 1` to compare messages I send to other people.

Unsurprisingly, the most common messages are very common one word ones.

From me:
```
text    count(text)
￼     1535   (I'm not sure if this is a specific emoji or any emoji)
Okay    446
ok      421
yeah    358
:(      234
okay    224
haha    220
great!  146
Yes     140
?       138
```

To me:
```
text    count(text)
￼     1772
yeah    358
Yeah    259
Yup     242
Ok      233
Yes     231
Yep     172
ok      159
yes     155
haha    154
```

Even though this is a tiny sample, you can start to get a sense of idiosyncracies in my texting habits. Unlike the rest of the world, I'm still sending typed frowny faces, typing out "okay", and sending a message that is just a question mark when I'm confused. I'm looking forward to doing more of this type of analysis as I continue!
