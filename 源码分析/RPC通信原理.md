1. 创建RPC协议接口RPCProtocal(版本号,方法)
```
   public interface RPCProtocol {
    long versionID = 666;

    void mkdirs(String path);
    }
```
2. 服务端实现接口(方法具体实现,创建RPC服务(服务器地址,端口号,通信协议))
```
public class RPCServer implements RPCProtocol {
    public static void main(String[] args) throws IOException {
        RPC.Server server = new RPC.Builder(new Configuration())
                .setBindAddress("localhost")
                .setPort(8888)
                .setProtocol(RPCServer.class)
                .setInstance(new NNServer())
                .build();
        System.out.println("server starts working");
        server.start();
    }

    @Override
    public void mkdirs(String path) {
        System.out.println("Server accept request from client" + path);
    }
}

```
3. 客户端获取服务端代理(服务器地址,端口号,通信协议)
```
public class HDFSClient {
    public static void main(String[] args) throws IOException {
        RPCProtocol client = RPC.getProxy(RPCProtocol.class, RPCProtocol.versionID, new InetSocketAddress("localhost", 8888), new Configuration());
        System.out.println("client starts working");
        client.mkdirs("/input  ");
    }
}
```
