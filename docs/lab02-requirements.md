Lab 02 – Requirements and Network Scenarios

Project: Mobile Parking Finder Emre Aracı

Functions

The app has these main functions:

Find the nearest parking with free spaces (read)
Search parking in an area on the map (read)
See details of a parking facility (read)
Reserve a space for 15 minutes (action)
Cancel a reservation (action)

I added reservation so the app has at least one operation that changes data on the server. Searching is just reading, so if the result is old it is annoying but nothing breaks. Reserving changes the free space count, so the app can only say "Reserved" after the server confirms it.

Offline decision: reservations are blocked when the phone is offline, not queued. If a reservation was queued and sent 10 minutes later, the space might already be taken and the user might be somewhere else. Blocking is simpler and less confusing.

Functional requirements
FR1: The app sends the current location with every search. Test: request includes lat, lon and time.
FR2: The server returns the 10 nearest parkings with free spaces, sorted by distance. Test: with test data, the list matches the expected order.
FR3: Every result shows how old it is ("updated 3 min ago"). Test: a 6 minute old cached result shows "6 min ago".
FR4: If the user moved more than 300 m since the result, the app searches again. Test: moving 400 m triggers a new search, 100 m does not.
FR5: Every reservation request has a unique request ID. Test: a request without ID is rejected.
FR6: If the same request ID comes twice, the server does the reservation only once. Test: sending the same request twice decreases free spaces by 1, not 2.
FR7: A reservation only succeeds if there is a free space at that moment. Test: 1 free space and 2 requests → one succeeds, one gets "full".
FR8: The app shows "Reserved" only after the server confirms. Test: if the response is blocked, the app shows "Pending", not "Reserved".
FR9: If the result is unknown, the app asks the server using the request ID. Test: after a lost response the app shows the real status from the server.
FR10: Reserve and cancel buttons are disabled when offline. Test: in airplane mode the buttons are disabled with a message.
FR11: The app keeps the last search result and reservation status in cache. Test: in airplane mode after restart, the last result is shown with its age.
Non-functional requirements
NFR1: Requests time out after 10 seconds. Test: a server delayed by 15 s causes a timeout at 10 s.
NFR2: Searches are retried at most 3 times. Test: server log shows at most 4 attempts.
NFR3: A retried reservation uses the same request ID. Test: both attempts in the log have the same ID.
NFR4: Results older than 5 minutes are marked as outdated. Test: a 6 minute old result is marked.
NFR5: Free spaces can never go below 0. Test: 20 parallel reservations on 5 spaces → 5 succeed, free spaces = 0.
NFR6: Search response is small (under 5 KB) to save wireless traffic. Test: measured size of a 10 result response.
NFR7: The app does not send location in the background. Test: no requests in the server log while the app is closed.
Network scenario matrix
#	Scenario	What the user sees	What happened on the server	Expected result
1	No connection before search	Last cached result + "no connection"	Nothing	Cached result shown with its age
2	Connection lost after search was sent	"Connection lost"	Search was done, answer not delivered	On reconnect, new search with new location
3	Search timeout	"Retrying..."	Unknown	Max 3 retries, then error + cached result
4	Late response, user moved 500 m	Result arrives late	Result is for old location	Result thrown away, new search sent
5	Reserve tapped while offline	Button disabled	Nothing	No reservation created
6	Reservation sent, response lost	"Checking status..."	Reservation created	App asks server by request ID → shows "Reserved", only 1 reservation exists
7	Reservation sent but never arrived	"Checking status..."	Nothing	Server says not found → "Not reserved, try again"
8	Reservation timeout	"Pending..."	Unknown	No success or fail shown until server answers
9	User taps Reserve twice / app retries	"Pending..."	First one already done	Same request ID, only 1 reservation
10	App opened offline with a reservation	"Reserved (confirmed 4 min ago)"	Reservation might have expired	Shows age; after 15 min shows "may be expired"
11	Reconnection	Status updates	Server has real state	App checks unknown requests and shows server's answer
12	Two users reserve the last space	One "Reserved", one "Full"	1 reservation, free spaces = 0	Exactly one success, never negative
