#[Working With TCP Sockets](https://github.com/da99/my_lib/blob/master/programming/linux/Working%20%20with%20%20TCP%20%20Sockets.pdf) - Jesse Storimer

## Socket

### Server Lifecycle
1. Create
2. Bind
3. Listen
4. Accept
5. Close

#### Create

    socket = Socket.new(:INET, :STREAM)
    
* `:INET` is IPv4 family of protocols.
* `:STREAM` is TCP. Use `:DGRAM` for UDP.
	
#### Bind

    # create a C struct to hold the address for listening
    addr = Socket.pack_sockaddr_in(4481, '0.0.0.0')
    
    # bind to it
    socket.bind(addr)

* Binds to a specific, agreed-upon port number which a client socket can then connect to.
* The code will exit immediately. The code doesn't yet do enough to actually listen for a connection.
* Port choice:
	* 0 - 1,024: Well known range
	* 49,000 - 65,535: Ephemeral range.
	* 1,024 <-> 49,000: Fair game. Check [IANA known ports](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.txt).
	
#### Listen

    # pass in the listening queue size
    socket.listen(Socket::SOMAXCONN)
    
* Code will still exit immediately.
* Max number of pending connections the server socket is willing to tolerate.
	* When full, will cause the client to raise `ECONNREFUSED`.
	* Current max is `Socket::SOMAXCONN`.

#### Accept

    server = Socket.new(:INET, :STREAM)
    addr = Socket.pack_sockaddr_in(4481, '0.0.0.0')
    server.bind(addr)
    server.listen(Socket::SOMAXCONN)
    
    # Accept a connection
    connection, remote_addr_info = server.accept
    
    print 'Local address: '
    p connection.local_address
    print 'Remote address: '
    p connection.remote_address
    
    <<-OUTPUT
      Local address: #<Addrinfo: 127.0.0.1:4481 TCP
      ￼Remote address: #<Addrinfo: 127.0.0.1:58164 TCP>
    OUTPUT
    
* Accept will block until a client connection arrives.

#### Close

* Closing the write stream will send a `EOF` to the remote server.
* Possible to close just one of the two-way communications by calling `connection.close_read` and `connection.close_write`.
* One can `#dup` a socket, and it will return a new socket instance using a new file descriptor under. Calling `#close` on the original will not close the duplicated socket's file description. However, calling `#shut_down` on the original will shut down communication on all copies of that socket.
	* When using `Process.fork`, implicitly all open file descriptors are `dup(2)`ed.


#### Server Ruby Wrappers

1.
   This:
   
      server = TCPServer.new(4481)
  replaces:
  
      server = Socket.new(:INET, :STREAM)
      addr = Socket.pack_sockaddr_in(4481, '0.0.0.')
      server.bind(addr)
      server.listen(Socket::SOMAXCONN)
      
  Caveats:
  1. `TCPServer#accept` returns only the connection, but not the addrinfo.
  1. By default, Ruby uses a listen queue of `5`. If you need more, call `TCPServer#listen` with a higher number after initialization.
      

1.
   To get both a IPv4 and IPv6 socket, listening on the same port, use:
   
      servers = Socket.tcp_server_sockets(4481)      
   
1.
   Use:
   
      server = TCPServer.new(4481)
      
      Socket.accept_loop(server) do |connection|
        # handle connection
        
        connection.close
      end
   To replace your `loop`, with the added benefit of allowing you to pass multiple listening sockets to it, and will accept _any_ connections from the given sockets.
   
   Caveats: You have to `#close` your own accepted connections.
   
1.
   Finally, use:
   
      Socket.tcp_server_loop(4481) do |connection|
        connection.close
      end
   To do all of the above, including listening on IPv6 sockets.
   
   Caveats: You have to `#close` your own accepted connections.

### Client Lifecycle

1. Create
2. Bind
3. Connect
4. Close

#### Bind

* Clients don't need to call bind most of the time, because they don't need to be accessible at a known port number.

#### Connect

* The only case where `#connect` returns successfully and not block forever until `ETIMEOUT` occurs is when the remote server has called `bind`, `listen`, and `accept`s the connection.

#### Client Ruby Wrappers

1.
      socket = TCPSocket.new('google.com', 80)
      
    or
    
      Socket.tcp('google.com', 80) do |connection|
        connection.write("GET / HTTP/1.1\r\n")
        connection.close
      end
      
    or
    
      # same as calling `TCPSocket.new` when used without a block
      client = Socket.tcp('google.com', 80)


## Reading

### Block on EOF
* `TCPSocket#read` will block when client does not send `EOF`.

### Block on No Data
* `TCPSocket#readpartial` will instead return data immediately, but will still block if no data is available.

### Non-Blocking
* `TCPSocket#read_nonblock` will never block.
	* Need to `rescue EAGAIN`, raised when no data were ready to be read.
		* Then call `IO.select([socket])` to `retry` with a blocking read.
	* Need to `rescue EOFError` and `break` the loop.
	
	
## Writing

### Block Until Done

* `TCPSocket#write` always take care of writing all of the data that you send it.

### Non-Blocking

* `TCPSocket#write_nonblock` will return (an integer) when it enters situations where it would block.
	* It is caller responsibility to take care of writing the rest of the data.
	* If underlying write(2) would still block, a `EAGAIN` error will be raised.
		* Use `IO.select` instead to detect when blocking situation is resolved.
		
## Accepting Clients
* `TCPServer#accept_nonblock` will raise `EAGAIN` when no client is ready.

## Connecting (surprise!)

* `Socket#connect_nonblock`:
   > Whereas the other methods either complete their operation or raise an appropriate exception, connect_nonblock leaves its operation in progress and raises an exception.
* If cannot make immediate connection to server, it lets operation continue in background and raise `EINPROGRESS`.
	* Raise `EALREADY` if a previous `connect_nonblock` is already in progress.
		* Use `IO.select(nil, [socket])` in its `rescue` to find out if the background connect has yet completed.
	* Raise `ECONNREFUSED` if remote host refused our connect.
	* Once inside `rescue EINPROGRESS`, use `IO.select` to check for writable sockets. Then issue another `socket.connect_nonblock` and if it raises `EISCONN`, then we have a successful connection. This is essentially what a _blocking_ `#connect` does.


## select(2) aka `IO.select`

* Nice Ruby wrapper `ready = IO.select(for_reading, for_writing, for_writing)`
	* Blocking call, and returns sockets that are ready to be read/written to/exception thrown.
	* Can also pass PORO to `IO.select`, so long as they respond to `#to_io` and return an `IO` object.
* Unable to monitor more than `FD_SETSIZE` number of file descriptors (1024 on most systems).
* Alternative to select(2), can use epoll(2) and kqueue(2) system calls to do similar things. See `nio4r` [gem](https://github.com/celluloid/nio4r) for examples.

## Nagle's Algorithm

* It is applied by all TCP connections by default.
* Most applicable to applications which don't do buffering and send very small amounts of data at a time, such as protocols like telnet.
* Every Ruby web server disables it.
	* To disable: `tcp_server.setsockopt(Socker::IPPROTO_TCP, Socket::TCP_NODELAY, 1)`.
	
## DNS Resolv

* Ruby GIL does understand blocking IO (e.g. a blocking `read`). MRI will release  the GIL and let other threads execute while being blocked by any blocking IO.
* MRI is, however, less forgiving when it comes to C extensions. The MRI GIL will cover the entirety of any library's C extension calls.
* Ruby uses a C extension for DNS lookups. As a result, a slow DNS will block the entire Ruby process.
* Solution: Use `resolv` library, a pure Ruby replacement for DNS lookups. Big win for multi-threaded environment.
* 
      require "resolv"         # the library
      require "resolv-replace" # monkeypatches `Socket` classes to use 'resolv'


## Useful Code Snippets

Creating self-signed SSL certificate ([source](https://github.com/ruby/ruby/blob/v2_1_1/lib/webrick/ssl.rb#L107-L109)):

    require "openssl"
    module Utils      
      # Creates a self-signed certificate with the given number of +bits+,
      # the issuer +cn+ and a +comment+ to be stored in the certificate.
    
      def create_self_signed_cert(bits, cn, comment)
        rsa = OpenSSL::PKey::RSA.new(bits){|p, n|
          case p
          when 0; $stderr.putc "."  # BN_generate_prime
          when 1; $stderr.putc "+"  # BN_generate_prime
          when 2; $stderr.putc "*"  # searching good prime,
                                    # n = #of try,
                                    # but also data from BN_generate_prime
          when 3; $stderr.putc "\n" # found good prime, n==0 - p, n==1 - q,
                                    # but also data from BN_generate_prime
          else;   $stderr.putc "*"  # BN_generate_prime
          end
        }
        cert = OpenSSL::X509::Certificate.new
        cert.version = 2
        cert.serial = 1
        name = OpenSSL::X509::Name.new(cn)
        cert.subject = name
        cert.issuer = name
        cert.not_before = Time.now
        cert.not_after = Time.now + (365*24*60*60)
        cert.public_key = rsa.public_key
    
        ef = OpenSSL::X509::ExtensionFactory.new(nil,cert)
        ef.issuer_certificate = cert
        cert.extensions = [
          ef.create_extension("basicConstraints","CA:FALSE"),
          ef.create_extension("keyUsage", "keyEncipherment"),
          ef.create_extension("subjectKeyIdentifier", "hash"),
          ef.create_extension("extendedKeyUsage", "serverAuth"),
          ef.create_extension("nsComment", comment),
        ]
        aki = ef.create_extension("authorityKeyIdentifier",
                                  "keyid:always,issuer:always")
        cert.add_extension(aki)
        cert.sign(rsa, OpenSSL::Digest::SHA1.new)
    
        return [ cert, rsa ]
      end
      module_function :create_self_signed_cert
    end


## Related Resources

1. How TCP connection handshake works: [A TCP "stuck" connection mystery](http://www.evanjones.ca/tcp-stuck-connection-mystery.html).

		To begin, let's briefly recap how TCP connections are established. The client begins by sending a SYN packet to the server. The server replies with a SYN-ACK packet and must remember some state about the connection. When the client receives the SYN-ACK, it considers its connection established, and replies with an ACK (and potentially some initial data). When the server gets the ACK, it considers its side of the connection to be established. However, what happens if a packet goes missing in this process? Well, if the SYN or SYN-ACK packets are lost, the client will retransmit the SYN a few times before giving up, returning an error to the application. If the ACK packet goes missing, then the server will resend the SYN-ACK, causing the client to retransmit the ACK. In other words: in nearly all cases, the client will either eventually have a connection, or will terminate with an error.

1. [Programming Ruby 1.9 - Socket Library](http://media.pragprog.com/titles/ruby3/app_socket.pdf)
