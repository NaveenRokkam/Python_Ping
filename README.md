# Python_Ping
Ping using Python
'''
Created on Apr 7, 2015

@author: Naveen.Rokkam
'''

from socket import * 
import os
import sys
import struct
import time
import select
import binascii
import code
from formatter import NullFormatter

ICMP_ECHO_REQUEST = 8
numberofpacketsreceived = 0
numberofpacketssent = 0
totalrtt = 0
rttmin = 999
rttmax = 0
error_messages = 'Error Message:'
g_type = 'type'
g_code = 'code'

def checksum(str):
    csum = 0
    countTo = (len(str) / 2) * 2
    count = 0
    while count < countTo:
        thisVal = ord(str[count+1]) * 256 + ord(str[count])
        csum = csum + thisVal
        csum = csum & 0xffffffffL
        count = count + 2
    if countTo < len(str):
        csum = csum + ord(str[len(str) - 1])
        csum = csum & 0xffffffffL
        
    csum = (csum >> 16) + (csum & 0xffff)
    csum = csum + (csum >> 16)
    answer = ~csum
    answer = answer & 0xffff
    answer = answer >> 8 | (answer << 8 & 0xff00)
    return answer

def error_msg(type, code):
    if (type == 3):
        if code == 0:
            msg = "Network Unreachable"
        if code == 1:
            msg = "Host Unreachable"
        if code == 2:
            msg = "Protocol Unreachable"
        if code == 3:
            msg = "Port Unreachable"
        if code == 4:
            msg = "Fragmentation needed and DF (Don't Fragment) set"
        if code == 5:
            msg = "Source route failed"
        if code == 6:
            msg = "Destination Network unknown"
        if code == 7:
            msg = "Destination Host unknown"
        if code == 8:
            msg = "Source Host isolated"
        if code == 9:
            msg = "Communication with Destination Network Administratively Prohibited"
        if code == 10:
            msg = "Communication with Destination Host Administratively Prohibited"
        if code == 11:
            msg = "Network Unreachable for Type Of Service"
        if code == 12:
            msg = "Host Unreachable for Type Of Service"
        if code == 13:
            msg = "Communication Administratively Prohibited by Filtering"
        if code == 14:
            msg = "Host Precedence Violation"
        if code == 15:
            msg = "Precedence Cutoff in Effect"
        
        return msg  


def receiveOnePing(mySocket, ID, timeout, destAddr):
    timeLeft = timeout
  
    while 1:
        startedSelect = time.time()
        whatReady = select.select([mySocket], [], [], timeLeft)
        howLongInSelect = (time.time() - startedSelect)
        if whatReady[0] == []: # Timeout
            return "Request timed out."
        
        timeReceived = time.time()
        recPacket, addr = mySocket.recvfrom(1024)
        global numberofpacketsreceived
        numberofpacketsreceived = numberofpacketsreceived+1
        icmpHeaderContent = recPacket[20:28]
        
        # Fetchs each detail from ICMP header
        type,code,checksum,packetID,sequence=struct.unpack("bbHHh",icmpHeaderContent)
        
        global g_type
        global g_code
        g_type = type
        g_code = code
              
        if (packetID==ID):
            bytesInDouble=struct.calcsize("d")
            timeSent=struct.unpack("d",recPacket[28:28 + bytesInDouble])[0]
            print ("Reply from " + str(destAddr) + ":" + " bytes=" + str(bytesInDouble) + " " )
            timediff = timeReceived - timeSent      
            return timediff
     
        timeLeft = timeLeft - howLongInSelect
        if timeLeft <= 0:
            return "Request timed out."
        
        
def sendOnePing(mySocket, destAddr, ID):
    # Header is type (8), code (8), checksum (16), id (16), sequence (16)
    myChecksum = 0
    # Make a dummy header with a 0 checksum
    # struct -- Interpret strings as packed binary data
    header = struct.pack("bbHHh", ICMP_ECHO_REQUEST, 0, myChecksum, ID, 1)
    data = struct.pack("d", time.time())
    # Calculate the checksum on the data and the dummy header.
    myChecksum = checksum(header + data)
    # Get the right checksum, and put in the header
    if sys.platform == 'darwin':
        # Convert 16-bit integers from host to network byte order
        myChecksum = htons(myChecksum) & 0xffff
    else:
        myChecksum = htons(myChecksum)
        
    header = struct.pack("bbHHh", ICMP_ECHO_REQUEST, 0, myChecksum, ID, 1)
    packet = header + data
    
    mySocket.sendto(packet, (destAddr, 1)) # AF_INET address must be tuple, not str
    # Both LISTS and TUPLES consist of a number of objects
    # which can be referenced by their position number within the object.
    
def doOnePing(destAddr, timeout):
    icmp = getprotobyname("icmp")
    # SOCK_RAW is a powerful socket type. For more details: http://sock-raw.org/papers/sock_raw
    try:
        mySocket= socket(AF_INET,SOCK_RAW, icmp)
    except socket.error, (error,msg):
        if errno == 1:
            msg = msg + 'Operation Not permitted'       
            raise socket.error(msg)
            
    myID = os.getpid() & 0xFFFF # Return the current process i
    sendOnePing(mySocket, destAddr, myID)
    global numberofpacketssent
    numberofpacketssent = numberofpacketssent+1
    
    delay = receiveOnePing(mySocket, myID, timeout, destAddr)
    mySocket.close()
    return delay

def ping(host, timeout=1):
# timeout=1 means: If one second goes by without a reply from the server,
# the client assumes that either the client's ping or the server's pong is lost
    dest = gethostbyname(host)
    print " "
    print "Pinging " + dest + " using Python:"

    #Send ping requests to a server separated by approximately one second
    while 1 :
        delay = doOnePing(dest, timeout)
        if (delay=="Request timed out."):
            print(delay)
            print(" ")
        else:
            delay=delay*1000
            print("rtt = " + str(delay) +" ms")
        time.sleep(1)# one second
        global totalrtt
        global rttmin
        global rttmax
        if (delay!="Request timed out."):
            if (delay<rttmin):
                rttmin=delay
            if (delay>=rttmax):
                rttmax=delay
            totalrtt=totalrtt+delay       
            
        # Print All Errors
        global g_type
        global g_code
        
        msg = error_msg(g_type, g_code)
        if (msg != None):
            print "Error Messages:" + msg   
            
        return delay

def clearvariables():
    global numberofpacketsreceived
    global numberofpacketssent
    global totalrtt
    global rttmin
    global rttmax
    
    numberofpacketsreceived = 0
    numberofpacketssent = 0
    totalrtt = 0
    rttmin = 999
    rttmax = 0
    
def run_test(host):
    print("*-----------------------------------------------------------------------------*")
    print("Pinging: "+host+" with 8 bytes of data")
    for i in range (0,100):
        ping(host)

    print("Ping Statistics for " +gethostbyname(host) + "::" + host)
    print("")
    print ("Packets: Sent = "+str(numberofpacketssent))
    print ("Packets: Received = "+str(numberofpacketsreceived))
    print ("Packets: lost =" +str(numberofpacketssent-numberofpacketsreceived))
    if((numberofpacketssent-numberofpacketsreceived)>=0):
        print ("Packets: lost% = "+str(((numberofpacketssent-numberofpacketsreceived)/numberofpacketssent)*100))
    else:
        print ("Packets: lost% = "+str(0))
    
    if (rttmin != 999): 
        print("Minimum RTT: " + str(rttmin)+" ms")
        print("Maximum RTT: " +str(rttmax)+" ms")
    else:           # When no packet is returned.
        print("Minimum RTT: Cannot be defined as all packets lost")
        print("Maximum RTT: Cannot be defined as all packets lost")
    if(numberofpacketsreceived != 0):
        print("Average RTT: " +str(totalrtt/numberofpacketsreceived)+" ms")
    else:
        print("Average RTT: Cannot be defined as No packets received")
    
#------------------------------------------------------------------------------------#

run_test("localhost")
clearvariables()


run_test("www.google.com")
clearvariables()

run_test("www.google.in")
clearvariables()

run_test("www.google.eu")
clearvariables()

run_test("amazon.cn")
clearvariables()
