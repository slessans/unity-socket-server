Unity Socket Server
===================

A socket server implementation for unity. Can be used with very few lines of code. 

Basically, can be used to allow clients to connects and send tokens. Here we define tokens as bytes in 
an arbitrary encoding, separated by an arbitrary string. 

For instance, you may want to get positional input of the form x,y,z. In this case you may choose
a separator like "&", so the stream may look like 1.0,1.0,1.0&2.0,1.2,.9&.

The socket reader will take care of connecting, reading the stream and splitting the tokens. Tou must provide
a delegate that will recieve notifications when a token is read. So in the case above it would recieve the strings
"1.0,1.0,1.0" and "2.0,1.2,.9".

The delegate method will most likely be invoked on a background thread, it is your job to implement thread-safety
(ususally can be done using a simple lock).

Creating the Server
===================

Use a persistent GameObject (one that will be alive for the full duration of the game). 
Attach a MonoBehaviour script. In this case, we will create a server that connects to 1 client and
reads in floats separated by "&".

Most likely, this MonoBehaviour will also be the delegate, in which case make it extend `SCL_IClientSocketHandlerDelegate`. More on this in a bit.

For this example well create a class called ExampleGameBehaviour:

    public class ExampleGameBehaviour : MonoBehaviour, SCL_IClientSocketHandlerDelegate {
        
      private SCL_SocketServer socketServer;
      private readonly object valueLock = new object();
      private float value;
      
      void Start() {
        // this will be filled in below
      }
      
      public void ClientSocketHandlerDidReadMessage(SCL_ClientSocketHandler handler, string message) {
       // this will be filled in below
      }
      
      void OnApplicationQuit() {
        // this will be filled in below
      }
      
      float GetValue() {
        // this will be filled in below
      }
        
    }
    
First, in the start method, we will make the server and start listening for connections. In this case we will be a bit 
verbose and show all the options:

    void Start() {
      SCL_IClientSocketHandlerDelegate clientSocketHandlerDelegate = this;
      int maxClients = 1;
      string separatorString = "&";
      int portNumber = 13000;
      Encoding encoding = Encoding.UTF8;
      
      this.socketServer = new SCL_SocketServer(
        clientSocketHandlerDelegate, maxClients, separatorString, portNumber, encoding);
      this.socketServer.StartListeningForConnections();
    }
    
The server will automatically stop listening for connections after `maxClients` connections have been made 
(in this case after one connection).

It is **very important** that you clean up the socket server. Fortunatley this is very easy:

    void OnApplicationQuit() {
      this.socketServer.Cleanup();
      this.socketServer = null;
    }
    
Now for the meat of the application. The delegate method. This gets called by the server when it recieves 
data from a client. It will pass the token read as a string (in the encoding passed in the constructor). The 
passed token does not include the separator string. In our case, we simply parse it as a float. **It is VERY
important to remember the delegate method gets called on another thread. Do not interact with the unity UI through
it and make sure any references to "this" are synchronized.**

    // this delegate method will be called on another thread, so use locks for synchronization
    public void ClientSocketHandlerDidReadMessage(SCL_ClientSocketHandler handler, string message)
    {
      try {
        // parse string value as float
        float val = float.Parse(message, CultureInfo.InvariantCulture);
        
        // if we get here, val is valid. update the instance variable. this must
        // be done using a lock
        lock(this.valueLock)
        {
          this.value = val;
        }
      } catch(FormatException) {
        Debug.Log ("Invalid token received: " + message);
      }
    }

That was easy! Now we simply finish it off by writing the `GetValue` method:

    float GetValue() {
      // this will be filled in below
      float val;
      lock(this.valueLock)
      {
        val = this.value;
      }
      return val;
    }
    
To use the value, you can call `GetValue` in FixedUpdate (or anywhere in your Unity game code for that matter):

    void FixedUpdate()
    {
        // dummy code:
        someObject.position.x = this.GetValue();
    }

Full Examples and Testing
==========================
Check out my other repository blank that is a [working example of this code](https://github.com/slessans/Simple-Unity-Socket-Server-Example/). It uses a socket server to accept
tokens of the form x,y,z& from 1 client and updates the position of a block accordingly.

For testing your own server out, it is often nice to be able to connect to it directly. Fire up the unity socket
server and connect using any TCP client. I have written a simple open-source one in Java that can be found here: [Simple Java Socket Client](https://github.com/slessans/Simple-Socket-Client-Java)


