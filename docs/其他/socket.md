# Socket

## 1.DataStream方式通讯

- 基于DataOutputStream的Socket客户端

```java
public static void main(String[] args){

    try {
        Socket socket = new Socket("127.0.0.1", 9999);
        OutputStream outputStream = socket.getOutputStream();
        DataOutputStream dos = new DataOutputStream(outputStream);
        dos.writeUTF("dafadsfads");
        dos.flush();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

- 基于DataInputStream的Socket服务端

```
public static void main(String[] args){
    try {
        ServerSocket serverSocket = new ServerSocket(9999);
        Socket accept = serverSocket.accept();
        InputStream inputStream = accept.getInputStream();
        DataInputStream dis = new DataInputStream(inputStream);
        String readUTF = dis.readUTF();
        System.out.println(readUTF);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

## 2.StreamWriter方式通讯

- 基于outputStreamWriter的Socket客户端

```java
public static void main(String[] args){

    try {
        Socket socket = new Socket("127.0.0.1", 9999);
        OutputStream outputStream = socket.getOutputStream();
        OutputStreamWriter outputStreamWriter = new OutputStreamWriter(outputStream);
        BufferedWriter bw = new BufferedWriter(outputStreamWriter);
        bw.write("dafadsfads");
        bw.newLine();
        bw.flush();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

- 基于InputStreamReader的Socket服务端

  ```
  public static void main(String[] args){
      try {
          ServerSocket serverSocket = new ServerSocket(9999);
          Socket accept = serverSocket.accept();
          InputStream inputStream = accept.getInputStream();
          InputStreamReader inputStreamReader = new InputStreamReader(inputStream);
          BufferedReader bufferedReader = new BufferedReader(inputStreamReader);
          String readLine = bufferedReader.readLine();
          System.out.println(readLine);
      } catch (IOException e) {
          e.printStackTrace();
      }
  }
  ```