import java.io.*;
import java.net.*;
import java.util.*;

public class FileStorageServer {
    private static final int PORT = 12345;
    private static final String STORAGE_DIR = "uploaded_files";

    public static void main(String[] args) {
        File dir = new File(STORAGE_DIR);
        if (!dir.exists()) {
            dir.mkdir(); // Create the storage directory if it doesn't exist
        }
        
        System.out.println("File storage server started...");
        
        try (ServerSocket serverSocket = new ServerSocket(PORT)) {
            while (true) {
                new ClientHandler(serverSocket.accept()).start();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static class ClientHandler extends Thread {
        private Socket socket;

        public ClientHandler(Socket socket) {
            this.socket = socket;
        }

        public void run() {
            try (DataInputStream in = new DataInputStream(socket.getInputStream());
                 DataOutputStream out = new DataOutputStream(socket.getOutputStream())) {
                
                String command = in.readUTF();
                switch (command) {
                    case "UPLOAD":
                        handleFileUpload(in);
                        break;
                    case "DOWNLOAD":
                        handleFileDownload(in, out);
                        break;
                    case "LIST":
                        handleFileList(out);
                        break;
                    default:
                        out.writeUTF("Invalid command");
                }
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                try {
                    socket.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }

        private void handleFileUpload(DataInputStream in) throws IOException {
            String fileName = in.readUTF();
            long fileSize = in.readLong();
            File file = new File(STORAGE_DIR, fileName);
            try (FileOutputStream fos = new FileOutputStream(file)) {
                byte[] buffer = new byte[4096];
                int read;
                long remaining = fileSize;

                while ((read = in.read(buffer, 0, (int) Math.min(buffer.length, remaining))) > 0) {
                    fos.write(buffer, 0, read);
                    remaining -= read;
                }
            }
            System.out.println("Received file: " + fileName);
        }

        private void handleFileDownload(DataInputStream in, DataOutputStream out) throws IOException {
            String fileName = in.readUTF();


            File file = new File(STORAGE_DIR, fileName);

            if (file.exists() && file.isFile()) {
                out.writeUTF("OK");
                out.writeLong(file.length());
                try (FileInputStream fis = new FileInputStream(file)) {
                    byte[] buffer = new byte[4096];
                    int read;

                    while ((read = fis.read(buffer)) > 0) {
                        out.write(buffer, 0, read);
                    }
                }
                System.out.println("Sent file: " + fileName);
            } else {
                out.writeUTF("File not found");
            }
        }

        private void handleFileList(DataOutputStream out) throws IOException {
            File dir = new File(STORAGE_DIR);
            String[] files = dir.list();
            if (files != null && files.length > 0) {
                out.writeUTF("OK");
                out.writeInt(files.length);
                for (String file : files) {
                    out.writeUTF(file);
                }
            } else {
                out.writeUTF("No files found");
            }
        }
    }
}

import java.io.*;
import java.net.*;
import java.util.Scanner;

public class FileStorageClient {
    private static final String SERVER_ADDRESS = "localhost";
    private static final int SERVER_PORT = 12345;

    public static void main(String[] args) {
        try (Socket socket = new Socket(SERVER_ADDRESS, SERVER_PORT)) {
            DataInputStream in = new DataInputStream(socket.getInputStream());
            DataOutputStream out = new DataOutputStream(socket.getOutputStream());
            Scanner scanner = new Scanner(System.in);

            while (true) {
                System.out.println("Choose an option:");
                System.out.println("1. Upload file");
                System.out.println("2. Download file");
                System.out.println("3. List files");
                System.out.println("4. Exit");
                System.out.print("Enter choice: ");
                int choice = scanner.nextInt();
                scanner.nextLine(); // Consume newline

                switch (choice) {
                    case 1:
                        System.out.print("Enter filename to upload: ");
                        String uploadFileName = scanner.nextLine();
                        uploadFile(out, uploadFileName);
                        break;

                    case 2:
                        System.out.print("Enter filename to download: ");
                        String downloadFileName = scanner.nextLine();
                        downloadFile(out, in, downloadFileName);
                        break;

                    case 3:
                        listFiles(out, in);
                        break;

                    case 4:
                        System.out.println("Exiting...");
                        return;

                    default:
                        System.out.println("Invalid choice");
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static void uploadFile(DataOutputStream out, String fileName) {
        try {
            File file = new File(fileName);
            if (!file.exists()) {
                System.out.println("File not found: " + fileName);
                return;
            }
            out.writeUTF("UPLOAD");
            out.writeUTF(file.getName());
            out.writeLong(file.length());


            try (FileInputStream fis = new FileInputStream(file)) {
                byte[] buffer = new byte[4096];
                int read;
                while ((read = fis.read(buffer)) > 0) {
                    out.write(buffer, 0, read);
                }
            }
            System.out.println("File uploaded: " + fileName);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static void downloadFile(DataOutputStream out, DataInputStream in, String fileName) {
        try {
            out.writeUTF("DOWNLOAD");
            out.writeUTF(fileName);
            String response = in.readUTF();
            if (!response.equals("OK")) {
                System.out.println(response);
                return;
            }
            long fileSize = in.readLong();
            File file = new File("downloaded_" + fileName);
            try (FileOutputStream fos = new FileOutputStream(file)) {
                byte[] buffer = new byte[4096];
                int read;
                long remaining = fileSize;

                while ((read = in.read(buffer, 0, (int) Math.min(buffer.length, remaining))) > 0) {
                    fos.write(buffer, 0, read);
                    remaining -= read;
                }
            }
            System.out.println("File downloaded: " + file.getName());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static void listFiles(DataOutputStream out, DataInputStream in) {
        try {
            out.writeUTF("LIST");
            String response = in.readUTF();
            if (response.equals("OK")) {
                int count = in.readInt();
                System.out.println("Files in storage:");
                for (int i = 0; i < count; i++) {
                    System.out.println("- " + in.readUTF());
                }
            } else {
                System.out.println(response);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
