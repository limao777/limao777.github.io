---
title: java操作文件的几种方法
date: 2017-10-20 09:18:33
category: java
tags:
- 文件
---

## 流式方式写入文件(使用字节)
```java
    /**
     * 流式方式写入文件(使用字节)
     * @param path
     */
    public static void writeFileUsingByte(String path){
        FileOutputStream fos = null;
        try {
            fos = new FileOutputStream(new File(path));
            fos.write("aaaaa\r\nbbb".getBytes());
        } catch (Exception e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }finally{
            try {
                fos.close();
            } catch (IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }      
    }
```

## 流式方式读取文件(使用字符)
```java
    /**
     * 流式方式读取文件(使用字符)
     * @param path
     */
    public static void readFileUsingByte(String path) {
        FileInputStream fis = null;
        try {
             fis = new FileInputStream(new File(path));
             byte []bytes = new byte[1024];
             int n = 0;
             while((n=fis.read(bytes))!=-1){
                 String s = new String(bytes,0,n);
                 System.out.println(s);
             }
            
        } catch (Exception e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }finally{
            try {
                fis.close();
            } catch (IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }
    }
```

## 流式方式读取文件(使用字符)
```java
    /**
     * 流式方式读取文件(使用字符)
     * @param path
     */
    public static void readFileUsingByte(String path) {
        FileInputStream fis = null;
        try {
             fis = new FileInputStream(new File(path));
             byte []bytes = new byte[1024];
             int n = 0;
             while((n=fis.read(bytes))!=-1){
                 String s = new String(bytes,0,n);
                 System.out.println(s);
             }
            
        } catch (Exception e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }finally{
            try {
                fis.close();
            } catch (IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }
    }
```

## 读出文件并写入到另外的文件中（使用字符）
```java
    /**
     * 读出文件并写入到另外的文件中（使用字符）
     * @param path
     */
    public static void dealFileUsingChar(String readPath, String writePath){
        FileReader fr = null;
        FileWriter wr = null;
        int n=0;
        try {
            fr = new FileReader(readPath);
            wr = new FileWriter(writePath);
            char []c=new char[1024];
            while((n=fr.read(c))!=-1){
                String s = new String(c,0,n);
                System.out.println(s);
                wr.write(s);
            }
        } catch (Exception e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }finally{
            try {
                fr.close();
                wr.close();
            } catch (IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }
    }
```

## 读出文件并写入到另外的文件中（使用缓冲）
```java
    /**
     * 读出文件并写入到另外的文件中（使用缓冲）
     * @param readPath
     * @param writePath
     */
    public static void dealFileUsingBufferedReader(String readPath, String writePath) {
        BufferedReader br = null;
        BufferedWriter wr = null;
       
        try {
            br = new BufferedReader(new FileReader(readPath));
            wr = new BufferedWriter(new FileWriter(writePath));
            String s="";
            while((s=br.readLine())!=null){
                System.out.println(s);
                wr.write(s);
            }
        } catch (Exception e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }finally{
            try {
                br.close();
                wr.close();
            } catch (IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }
    }

```
