# Packet Pacer

**UDP (User Datagram Protocol)**- is a networking protocol based in the transport layer, it facilitates faster communication in comparison to TCP/IP, but causes packets to be lost in its way. Its applications include audio/video communication, DNS lookup etc.

### PROBLEM STATEMENT
We are provided with a UDPsocket and a testinit function that initiates multiple socket readers spawned at once using the go channels, as well as a UDP writer that writes into each one of the created ports. Our job is to significantly increase the performance of the UDP network writer significantly.
### DESCRIPTION
The BenchmarkRawUDP function provided to us uses the net.ListenUDP method to create a listener and use conn.writer to write to the various reader connected at various ports, also something to be noted that the UDPsocket used to create readers are made using the underlying syscall library in Go, it uses AF_INET protocol to create the socket, which is then binded to a port again using syscall method. The testinit function is already in its reduced form and cannot be edited according to the instructions provided.
Our approach would be to to use underlying low-level methods such as the syscall library to create a writer instead of the “net” library provided and make use of the asynchronous methods available to us.

### SOLUTION  
We implement within the BenchmarkSample function a UDP listener using the syscall method we specify AF_INET as the protocol, SOCK_DGRAM which specifies a datagram-based connection and PROTO_IP, to specify that we transfer IP packets. We then bind this socket to the reader’s IP address and a test port.
We then receive the ports on which the reader has been created, Now are we use go channels to achieve this where each packet is sent simultaneously to the reader ports, this decreases the overall time taken and significantly increases the performance of the UDP network writer, we have declared wait groups equal to the readers which prevents the functions from returning/ending before all go channels have completed their tasks. 
We must mention that the buffer containing the message/packet (getTestMsg function) to be transferred cannot be moved out of the loop according to the instructions, its position is still maintained. ( However, if we were allowed to move this outside the loop i.e. Same message is sent to every listener , then the speed increases drastically)

```
func BenchmarkSample(b *testing.B) {
    b.StopTimer()
    // Do something here
    testPort := 40101
    fd, err := syscall.Socket(syscall.AF_INET, syscall.SOCK_DGRAM, syscall.IPPROTO_UDP)
    if err != nil {
        log.Fatalf("Failed to create socket: %v", err)
    }
    defer syscall.Close(fd)

    addr := [4]byte{127, 0, 0, 1}
    err = syscall.Bind(fd, &syscall.SockaddrInet4{
        Port: testPort,
        Addr: addr,
    })
    if err != nil {
        b.Fatal(err)
    }

    ports, readChan, closeChan, err := testInit(readersCount, false) // DO NOT EDIT THIS LINE
    if err != nil {
        b.Fatal(err)
    }
    _ = readChan

    writer := func() {
        var wg sync.WaitGroup
        wg.Add(readersCount)
        for i := 0; i < readersCount; i++ {
            go func(i int) {
                defer wg.Done()
                buf := getTestMsg()// DO NOT EDIT THIS LINE

                err := syscall.Sendto(fd, buf, 0, &syscall.SockaddrInet4{
                    Port: ports[i],
                    Addr: addr,
                })
                if err != nil {
                    b.Fatal(err)
                }
            }(i)
        }

        // End of code that you are permitted to modify
        waitForReaders(readChan, b)
    }

    // Sequential test
    b.StartTimer()
    for i := 0; i < b.N; i++ {
        writer()
    }
    b.StopTimer()

    close(closeChan)
}

```

With this approach of using low level implementations and go channels we were able to **almost double the performance of the UDP network writer.**

<img width="1012" height="229" alt="image" src="https://github.com/user-attachments/assets/760bcda5-98c2-4bef-880b-64e13f708936" />


<img width="1012" height="202" alt="image" src="https://github.com/user-attachments/assets/0003cde3-86f7-47dc-8aee-1452971c45ea" />

we have successfully doubled the number of iterations, the allocations per operation suggest that the program is using more system memory than the original implementation. Probably in the form of the system buffers, since using Go channels can increase the overhead.
However, there may be cases where we can see, a difference of +/-15 iterations from the value of doubled iterations. Performance may improve with better processors.

### Future Scope
The solution can be improved by using protocols like AF_PACKET or XDP but these work on network layers and not network ports. We would have to create a socket of type SOCKET_RAW and build the whole package configuration including the source and destination port, source and destination IP, mac address along with the ethernet header. 
The challenge in implementing this solution however is that the Ports received from the UDP reader sockets will need to be dismembered into network layers to write into them, also for every port received the packet needs to be rebuilt which takes considerable amount of time.
Also since since the byte size is set to 1500 , which is the MTU(max transferrable unit) for `ethernet` header which needs to be provided while using raw_socket or AF_Packet techniques, it would hit a `message too long’ error since the raw socket requires extra 60/70 bytes at least to store configuration information.
Hence under the given conditions the provided solution is the most optimized


