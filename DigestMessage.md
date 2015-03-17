# Howto digest messages in Java #

```
    // Obtain a message digester (recommended to store it for further use):
    MessageDigest messageDigest = null;
    try {
        messageDigest = MessageDigest.getInstance( "SHA-256" );
    }
    catch ( final NoSuchAlgorithmException nsae ) {
        throw new RuntimeException( "This should never happen!" );
    }
    
    // Message to be digested
    final String message = "Hello World!";
    
    // Calculate digest bytes:
    final byte[] messageBytes = messageDigest.digest( message.getBytes() );
    
    // Convert it to hex string:
    final StringBuilder sb = new StringBuilder( messageBytes.length * 2 );
    for ( int i = 0; i < messageBytes.length; i++ )
        sb.append( Integer.toHexString( ( messageBytes[ i ] & 0xff ) >> 4 ) ).append( Integer.toHexString( messageBytes[ i ] & 0x0f ) );
    
    // Here sb contains the hex string of the digest of the message
```