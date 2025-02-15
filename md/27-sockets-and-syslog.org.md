Chapter 27. Sockets and Syslog
==============================

Basic Networking
----------------

In several earlier chapters of this book, we have discussed services
that operate over a network. Two examples are client/server databases
and web services. When the need arises to devise a new protocol, or to
communicate with a protocol that doesn\'t have an existing helper
library in Haskell, you\'ll need to use the lower-level networking tools
in the Haskell library.

In this chapter, we will discuss these lower-level tools. Network
communication is a broad topic with entire books devoted to it. In this
chapter, we will show you how to use Haskell to apply low-level network
knowledge you already have.

Haskell\'s networking functions almost always correspond directly to
familiar C function calls. As most other languages also layer atop C,
you should find this interface familiar.

Communicating with UDP
----------------------

UDP breaks data down into packets. It does not ensure that the data
reaches its destination, or reaches it only once. It does use
checksumming to ensure that packets that arrive have not been corrupted.
UDP tends to be used in applications that are performance- or
latency-sensitive, in which each individual packet of data is less
important than the overall performance of the system. It may also be
used where the TCP behavior isn\'t the most efficient, such as ones that
send short, discrete messages. Examples of systems that tend to use UDP
include audio and video conferencing, time synchronization,
network-based filesystems, and logging systems.

### UDP Client Example: syslog

The traditional Unix syslog service allows programs to send log messages
over a network to a central server that records them. Some programs are
quite performance-sensitive, and may generate a large volume of
messages. In these programs, it could be more important to have the
logging impose a minimal performance overhead than to guarantee every
message is logged. Moreover, it may be desirable to continue program
operation even if the logging server is unreachable. For this reason,
UDP is one of the protocols supported by syslog for the transmission of
log messages. The protocol is simple and we present a Haskell
implementation of a client here.

``` {.example}
import Data.Bits
import Network.Socket
import Network.BSD
import Data.List
import SyslogTypes

data SyslogHandle =
    SyslogHandle {slSocket :: Socket,
                  slProgram :: String,
                  slAddress :: SockAddr}

openlog :: HostName             -- ^ Remote hostname, or localhost
        -> String               -- ^ Port number or name; 514 is default
        -> String               -- ^ Name to log under
        -> IO SyslogHandle      -- ^ Handle to use for logging
openlog hostname port progname =
    do -- Look up the hostname and port.  Either raises an exception
       -- or returns a nonempty list.  First element in that list
       -- is supposed to be the best option.
       addrinfos <- getAddrInfo Nothing (Just hostname) (Just port)
       let serveraddr = head addrinfos

       -- Establish a socket for communication
       sock <- socket (addrFamily serveraddr) Datagram defaultProtocol

       -- Save off the socket, program name, and server address in a handle
       return $ SyslogHandle sock progname (addrAddress serveraddr)

syslog :: SyslogHandle -> Facility -> Priority -> String -> IO ()
syslog syslogh fac pri msg =
    sendstr sendmsg
    where code = makeCode fac pri
          sendmsg = "<" ++ show code ++ ">" ++ (slProgram syslogh) ++
                    ": " ++ msg

          -- Send until everything is done
          sendstr :: String -> IO ()
          sendstr [] = return ()
          sendstr omsg = do sent <- sendTo (slSocket syslogh) omsg
                                    (slAddress syslogh)
                            sendstr (genericDrop sent omsg)

closelog :: SyslogHandle -> IO ()
closelog syslogh = sClose (slSocket syslogh)

{- | Convert a facility and a priority into a syslog code -}
makeCode :: Facility -> Priority -> Int
makeCode fac pri =
    let faccode = codeOfFac fac
        pricode = fromEnum pri
        in
          (faccode `shiftL` 3) .|. pricode
```

This also requires `SyslogTypes.hs`, shown here:

``` {.example}
module SyslogTypes where
{- | Priorities define how important a log message is. -}

data Priority =
            DEBUG                   -- ^ Debug messages
          | INFO                    -- ^ Information
          | NOTICE                  -- ^ Normal runtime conditions
          | WARNING                 -- ^ General Warnings
          | ERROR                   -- ^ General Errors
          | CRITICAL                -- ^ Severe situations
          | ALERT                   -- ^ Take immediate action
          | EMERGENCY               -- ^ System is unusable
                    deriving (Eq, Ord, Show, Read, Enum)

{- | Facilities are used by the system to determine where messages
are sent. -}

data Facility =
              KERN                      -- ^ Kernel messages
              | USER                    -- ^ General userland messages
              | MAIL                    -- ^ E-Mail system
              | DAEMON                  -- ^ Daemon (server process) messages
              | AUTH                    -- ^ Authentication or security messages
              | SYSLOG                  -- ^ Internal syslog messages
              | LPR                     -- ^ Printer messages
              | NEWS                    -- ^ Usenet news
              | UUCP                    -- ^ UUCP messages
              | CRON                    -- ^ Cron messages
              | AUTHPRIV                -- ^ Private authentication messages
              | FTP                     -- ^ FTP messages
              | LOCAL0
              | LOCAL1
              | LOCAL2
              | LOCAL3
              | LOCAL4
              | LOCAL5
              | LOCAL6
              | LOCAL7
                deriving (Eq, Show, Read)

facToCode = [
                       (KERN, 0),
                       (USER, 1),
                       (MAIL, 2),
                       (DAEMON, 3),
                       (AUTH, 4),
                       (SYSLOG, 5),
                       (LPR, 6),
                       (NEWS, 7),
                       (UUCP, 8),
                       (CRON, 9),
                       (AUTHPRIV, 10),
                       (FTP, 11),
                       (LOCAL0, 16),
                       (LOCAL1, 17),
                       (LOCAL2, 18),
                       (LOCAL3, 19),
                       (LOCAL4, 20),
                       (LOCAL5, 21),
                       (LOCAL6, 22),
                       (LOCAL7, 23)
           ]

codeToFac = map (\(x, y) -> (y, x)) facToCode

{- | We can't use enum here because the numbering is discontiguous -}
codeOfFac :: Facility -> Int
codeOfFac f = case lookup f facToCode of
                Just x -> x
                _ -> error $ "Internal error in codeOfFac"

facOfCode :: Int -> Facility
facOfCode f = case lookup f codeToFac of
                Just x -> x
                _ -> error $ "Invalid code in facOfCode"
```

With `ghci`, you can send a message to a local syslog server. You can
use either the example syslog server presented in this chapter, or an
existing syslog server like you would typically find on Linux or other
POSIX systems. Note that most of these disable the UDP port by default
and you may need to enable UDP before your vendor-supplied syslog daemon
will display received messages.

If you were sending a message to a syslog server on the local system,
you might use a command such as this:

``` {.screen}
ghci> :load syslogclient.hs
[1 of 2] Compiling SyslogTypes      ( SyslogTypes.hs, interpreted )
[2 of 2] Compiling Main             ( syslogclient.hs, interpreted )
Ok, modules loaded: SyslogTypes, Main.
ghci> h <- openlog "localhost" "514" "testprog"
Loading package parsec-2.1.0.0 ... linking ... done.
Loading package network-2.1.0.0 ... linking ... done.
ghci> syslog h USER INFO "This is my message"
ghci> closelog h
```

### UDP Syslog Server

UDP servers will bind to a specific port on the server machine. They
will accept packets directed to that port and process them. Since UDP is
a stateless, packet-oriented protocol, programmers normally use a call
such as `recvFrom` to receive both the data and information about the
machine that sent it, which is used for sending back a response.

``` {.example}
import Data.Bits
import Network.Socket
import Network.BSD
import Data.List

type HandlerFunc = SockAddr -> String -> IO ()

serveLog :: String              -- ^ Port number or name; 514 is default
         -> HandlerFunc         -- ^ Function to handle incoming messages
         -> IO ()
serveLog port handlerfunc = withSocketsDo $
    do -- Look up the port.  Either raises an exception or returns
       -- a nonempty list.
       addrinfos <- getAddrInfo
                    (Just (defaultHints {addrFlags = [AI_PASSIVE]}))
                    Nothing (Just port)
       let serveraddr = head addrinfos

       -- Create a socket
       sock <- socket (addrFamily serveraddr) Datagram defaultProtocol

       -- Bind it to the address we're listening to
       bindSocket sock (addrAddress serveraddr)

       -- Loop forever processing incoming data.  Ctrl-C to abort.
       procMessages sock
    where procMessages sock =
              do -- Receive one UDP packet, maximum length 1024 bytes,
                 -- and save its content into msg and its source
                 -- IP and port into addr
                 (msg, _, addr) <- recvFrom sock 1024
                 -- Handle it
                 handlerfunc addr msg
                 -- And process more messages
                 procMessages sock

-- A simple handler that prints incoming packets
plainHandler :: HandlerFunc
plainHandler addr msg =
    putStrLn $ "From " ++ show addr ++ ": " ++ msg
```

You can run this in `ghci`. A call to `serveLog "1514" plainHandler`
will set up a UDP server on port 1514 that will use `plainHandler` to
print out every incoming UDP packet on that port. Ctrl-C will terminate
the program.

::: {.NOTE}
In case of problems

Getting `bind: permission denied` when testing this? Make sure you use a
port number greater than 1024. Some operating systems only allow the
`root` user to bind to ports less than 1024.
:::

Communicating with TCP
----------------------

TCP is designed to make data transfer over the Internet as reliable as
possible. TCP traffic is a stream of data. While this stream gets broken
up into individual packets by the operating system, the packet
boundaries are neither known nor relevant to applications. TCP
guarantees that, if traffic is delivered to the application at all, that
it has arrived intact, unmodified, exactly once, and in order.
Obviously, things such as a broken wire can cause traffic to not be
delivered, and no protocol can overcome those limitations.

This brings with it some tradeoffs compared with UDP. First of all,
there are a few packets that must be sent at the start of the TCP
conversation to establish the link. For very short conversations, then,
UDP would have a performance advantage. Also, TCP tries very hard to get
data through. If one end of a conversation tries to send data to the
remote, but doesn\'t receive an acknowledgment back, it will
periodically re-transmit the data for some time before giving up. This
makes TCP robust in the face of dropped packets. However, it also means
that TCP is not the best choice for real-time protocols that involve
things such as live audio or video.

### Handling Multiple TCP Streams

With TCP, connections are stateful. That means that there is a dedicated
logical \"channel\" between a client and server, rather than just
one-off packets as with UDP. This makes things easy for client
developers. Server applications almost always will want to be able to
handle more than one TCP connection at once. How then to do this?

On the server side, you will first create a socket and bind to a port,
just like UDP. Instead of repeatedly listening for data from any
location, your main loop will be around the `accept` call. Each time a
client connects, the server\'s operating system allocates a new socket
for it. So we have the *master* socket, used only to listen for incoming
connections, and never to transmit data. We also have the potential for
multiple *child* sockets to be used at once, each corresponding to a
logical TCP conversation.

In Haskell, you will usually use `forkIO` to create a separate
lightweight thread to handle each conversation with a child. Haskell has
an efficient internal implementation of this that performs quite well.

### TCP Syslog Server

Let\'s say that we wanted to reimplement syslog using TCP instead of
UDP. We could say that a single message is defined not by being in a
single packet, but is ended by a trailing newline character `'\n'`. Any
given client could send 0 or more messages to the server using a given
TCP connection. Here\'s how we might write that.

``` {.example}
import Data.Bits
import Network.Socket
import Network.BSD
import Data.List
import Control.Concurrent
import Control.Concurrent.MVar
import System.IO

type HandlerFunc = SockAddr -> String -> IO ()

serveLog :: String              -- ^ Port number or name; 514 is default
         -> HandlerFunc         -- ^ Function to handle incoming messages
         -> IO ()
serveLog port handlerfunc = withSocketsDo $
    do -- Look up the port.  Either raises an exception or returns
       -- a nonempty list.
       addrinfos <- getAddrInfo
                    (Just (defaultHints {addrFlags = [AI_PASSIVE]}))
                    Nothing (Just port)
       let serveraddr = head addrinfos

       -- Create a socket
       sock <- socket (addrFamily serveraddr) Stream defaultProtocol

       -- Bind it to the address we're listening to
       bindSocket sock (addrAddress serveraddr)

       -- Start listening for connection requests.  Maximum queue size
       -- of 5 connection requests waiting to be accepted.
       listen sock 5

       -- Create a lock to use for synchronizing access to the handler
       lock <- newMVar ()

       -- Loop forever waiting for connections.  Ctrl-C to abort.
       procRequests lock sock

    where
          -- | Process incoming connection requests
          procRequests :: MVar () -> Socket -> IO ()
          procRequests lock mastersock =
              do (connsock, clientaddr) <- accept mastersock
                 handle lock clientaddr
                    "syslogtcpserver.hs: client connected"
                 forkIO $ procMessages lock connsock clientaddr
                 procRequests lock mastersock

          -- | Process incoming messages
          procMessages :: MVar () -> Socket -> SockAddr -> IO ()
          procMessages lock connsock clientaddr =
              do connhdl <- socketToHandle connsock ReadMode
                 hSetBuffering connhdl LineBuffering
                 messages <- hGetContents connhdl
                 mapM_ (handle lock clientaddr) (lines messages)
                 hClose connhdl
                 handle lock clientaddr
                    "syslogtcpserver.hs: client disconnected"

          -- Lock the handler before passing data to it.
          handle :: MVar () -> HandlerFunc
          -- This type is the same as
          -- handle :: MVar () -> SockAddr -> String -> IO ()
          handle lock clientaddr msg =
              withMVar lock
                 (\a -> handlerfunc clientaddr msg >> return a)

-- A simple handler that prints incoming packets
plainHandler :: HandlerFunc
plainHandler addr msg =
    putStrLn $ "From " ++ show addr ++ ": " ++ msg
```

For our `SyslogTypes` implementation, see [the section called \"UDP
Client Example:
syslog\"](27-sockets-and-syslog.org::*UDP Client Example: syslog)

Let\'s look at this code. Our main loop is in `procRequests`, where we
loop forever waiting for new connections from clients. The `accept` call
blocks until a client connects. When a client connects, we get a new
socket and the address of the client. We pass a message to the handler
about that, then use `forkIO` to create a thread to handle the data from
that client. This thread runs `procMessages`.

When dealing with TCP data, it\'s often convenient to convert a socket
into a Haskell `Handle`. We do so here, and explicitly set the
buffering---an important point for TCP communication. Next, we set up
lazy reading from the socket\'s `Handle`. For each incoming line, we
pass it to `handle`. After there is no more data---because the remote
end has closed the socket---we output a message about that.

Since we may be handling multiple incoming messages at once, we need to
ensure that we\'re not writing out multiple messages at once in the
handler. That could result in garbled output. We use a simple lock to
serialize access to the handler, and write a simple `handle` function to
handle that.

You can test this with the client we\'ll present next, or you can even
use the `telnet` program to connect to this server. Each line of text
you send to it will be printed on the display by the server. Let\'s try
it out:

``` {.screen}
ghci> :load syslogtcpserver.hs
[1 of 1] Compiling Main             ( syslogtcpserver.hs, interpreted )
Ok, modules loaded: Main.
ghci> serveLog "10514" plainHandler
Loading package parsec-2.1.0.0 ... linking ... done.
Loading package network-2.1.0.0 ... linking ... done.
```

At this point, the server will begin listening for connections at port
10514. It will not appear to be doing anything until a client connects.
We could use telnet to connect to the server:

``` {.screen}
~$ telnet localhost 10514
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Test message
^]
telnet> quit
Connection closed.
```

Meanwhile, in our other terminal running the TCP server, you\'ll see
something like this:

``` {.screen}
From 127.0.0.1:38790: syslogtcpserver.hs: client connected
From 127.0.0.1:38790: Test message
From 127.0.0.1:38790: syslogtcpserver.hs: client disconnected
```

This shows that a client connected from port 38790 on the local machine
(127.0.0.1). After it connected, it sent one message, and disconnected.
When you are acting as a TCP client, the operating system assigns an
unused port for you. This port number will usually be different each
time you run the program.

### TCP Syslog Client

Now, let\'s write a client for our TCP syslog protocol. This client will
be similar to the UDP client, but there are a couple of changes. First,
since TCP is a streaming protocol, we can send data using a `handle`
rather than using the lower-level socket operations. Secondly, we no
longer need to store the destination address in the `SyslogHandle` since
we will be using `connect` to establish the TCP connection. Finally, we
need a way to know where one message ends and the next begins. With UDP,
that was easy because each message was a discrete logical packet. With
TCP, we\'ll just use the newline character `'\n'` as the end-of-message
marker, though that means that no individual message may contain the
newline. Here\'s our code:

``` {.example}
import Data.Bits
import Network.Socket
import Network.BSD
import Data.List
import SyslogTypes
import System.IO

data SyslogHandle =
    SyslogHandle {slHandle :: Handle,
                  slProgram :: String}

openlog :: HostName             -- ^ Remote hostname, or localhost
        -> String               -- ^ Port number or name; 514 is default
        -> String               -- ^ Name to log under
        -> IO SyslogHandle      -- ^ Handle to use for logging
openlog hostname port progname =
    do -- Look up the hostname and port.  Either raises an exception
       -- or returns a nonempty list.  First element in that list
       -- is supposed to be the best option.
       addrinfos <- getAddrInfo Nothing (Just hostname) (Just port)
       let serveraddr = head addrinfos

       -- Establish a socket for communication
       sock <- socket (addrFamily serveraddr) Stream defaultProtocol

       -- Mark the socket for keep-alive handling since it may be idle
       -- for long periods of time
       setSocketOption sock KeepAlive 1

       -- Connect to server
       connect sock (addrAddress serveraddr)

       -- Make a Handle out of it for convenience
       h <- socketToHandle sock WriteMode

       -- We're going to set buffering to BlockBuffering and then
       -- explicitly call hFlush after each message, below, so that
       -- messages get logged immediately
       hSetBuffering h (BlockBuffering Nothing)

       -- Save off the socket, program name, and server address in a handle
       return $ SyslogHandle h progname

syslog :: SyslogHandle -> Facility -> Priority -> String -> IO ()
syslog syslogh fac pri msg =
    do hPutStrLn (slHandle syslogh) sendmsg
       -- Make sure that we send data immediately
       hFlush (slHandle syslogh)
    where code = makeCode fac pri
          sendmsg = "<" ++ show code ++ ">" ++ (slProgram syslogh) ++
                    ": " ++ msg

closelog :: SyslogHandle -> IO ()
closelog syslogh = hClose (slHandle syslogh)

{- | Convert a facility and a priority into a syslog code -}
makeCode :: Facility -> Priority -> Int
makeCode fac pri =
    let faccode = codeOfFac fac
        pricode = fromEnum pri
        in
          (faccode `shiftL` 3) .|. pricode
```

We can try it out under `ghci`. If you still have the TCP server running
from earlier, your session might look something like this:

``` {.screen}
ghci> :load syslogtcpclient.hs
Loading package base ... linking ... done.
[1 of 2] Compiling SyslogTypes      ( SyslogTypes.hs, interpreted )
[2 of 2] Compiling Main             ( syslogtcpclient.hs, interpreted )
Ok, modules loaded: Main, SyslogTypes.
ghci> openlog "localhost" "10514" "tcptest"
Loading package parsec-2.1.0.0 ... linking ... done.
Loading package network-2.1.0.0 ... linking ... done.
ghci> sl <- openlog "localhost" "10514" "tcptest"
ghci> syslog sl USER INFO "This is my TCP message"
ghci> syslog sl USER INFO "This is my TCP message again"
ghci> closelog sl
```

Over on the server, you\'ll see something like this:

``` {.screen}
From 127.0.0.1:46319: syslogtcpserver.hs: client connected
From 127.0.0.1:46319: <9>tcptest: This is my TCP message
From 127.0.0.1:46319: <9>tcptest: This is my TCP message again
From 127.0.0.1:46319: syslogtcpserver.hs: client disconnected
```

The `<9>` is the priority and facility code being sent along, just as it
was with UDP.
