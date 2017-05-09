title: 读写hdfs代码
date: 2017-05-09 15:41:36
tags: [hadoop]
category: [hadoop]
---

## 1) 读取hdfs文件
<!--more-->
```
// FileSystemCat.java:

package main;


import java.io.IOException;
import java.io.InputStream;
import java.net.URI;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IOUtils;

public class FileSystemCat 
{
	public static void main( String[] args )
	{
		String uri = args[0];
		Configuration conf = new Configuration();
		FileSystem fs = null;
		
		try 
		{
			fs = FileSystem.get( URI.create(uri), conf );
		} catch (IOException e1) 
		{
			e1.printStackTrace();
		}
		
		InputStream in = null;
		
		try
		{
			in = fs.open( new Path(uri) );
			IOUtils.copyBytes( in, System.out, 4096, false );
		}
		catch( Exception e )
		{
			
		}
		finally
		{
			IOUtils.closeStream(in);
		}
		
		return;
	}

}
```

## 2) 将本地文件复制到hadoop文件系统
```
// FileCopyWithProgress.java:
package main;
 

import java.io.BufferedInputStream;  
import java.io.FileInputStream;  
import java.io.InputStream;  
import java.io.OutputStream;  
import java.net.URI;  
  
import org.apache.hadoop.conf.Configuration;  
import org.apache.hadoop.fs.FileSystem;  
import org.apache.hadoop.fs.Path;  
import org.apache.hadoop.io.IOUtils;  
import org.apache.hadoop.util.Progressable;  
  
public class FileCopyWithProgress {  
  
    /** 
     * @param args 
     */  
    public static void main(String[] args) throws Exception{  
        // TODO Auto-generated method stub  
        String localSrc = args[0];  
        String dst = args[1];  
        InputStream in = new BufferedInputStream(new FileInputStream(localSrc));  
        Configuration conf = new Configuration();  
        FileSystem fs = FileSystem.get(URI.create(dst), conf);  
        OutputStream out = fs.create(new Path(dst), new Progressable(){  
            public void progress(){  
                System.out.print(".");  
            }  
        });  
          
        IOUtils.copyBytes(in, out, 4096, true);  
    }  
  
}  

$ hadoop FileCopyWithProgress input/1.txt hdfs://localhost/user/2.txt
.......
上面的点为显示进度
```

## 3) URLCat.java
```
package main;

import java.io.IOException;
import java.io.InputStream;
import java.net.MalformedURLException;
import java.net.URL;

import org.apache.hadoop.fs.FsUrlStreamHandlerFactory;
import org.apache.hadoop.io.IOUtils;

public class URLCat 
{
	static
	{
		URL.setURLStreamHandlerFactory( new FsUrlStreamHandlerFactory() );
		
	}
	
	public static void main( String[] args )
	{
		InputStream in = null;
		
		try
		{
			
			in = new URL(args[0]).openStream();
			IOUtils.copyBytes( in, System.out, 4096, false );
		}
		catch( Exception e )
		{
			
		}
		finally
		{
			IOUtils.closeStream(in);
		}
		
		return;
	}

}
```


