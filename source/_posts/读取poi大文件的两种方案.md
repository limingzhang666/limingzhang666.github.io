---
title: 读取poi大文件的两种方案
copyright: true
comments: true
related_posts: true
date: 2021-03-28 15:51:16
tags: 读取大文件
categories: poi
---


## 背景
最近有个业务需求，功能是：要通过excel 装载大量数据并上传至系统，excel的格式是.xlsx,后台需要解析excel的每条数据，进行一系列 有效性以及权限等校验。
验证通过的需要落数据库， 验证不通过的需要 生成一个异常数据的excel，将每条数据的错误原因 备注在 源数据的最后一列 。

其实举个例子，比如 5w条数据放在excel，excel的大小大概是 2M左右，在放在List里面不算大，可是使用用户模式POI去一次性解析放在List<T>中，这个过程 需要的内存肯定是 20M 往上， 10倍都是保守估计， 所以不要看excel文件不大，解析可是非常耗内存的， 可能是 xlsx的功能太多了，文件格式 太复杂了

所以建议当数据量大的时候，使用.csv的文件去上传， 因为csv 可以使用excel打开，也可以用文本打开，单元格数据用的 逗号隔开的， 只需要用splitBy 逗号就行了，很便捷 



## 方案

其实如果数据量不大的情况下，我们 可以使用简单的poi 将excel 一次性加载至内存中，然后逐条数据一次解析验证 。

可是当 excel 的数据量特别大的时候，就有可能发生OOM的情况，系统直接就卡死了。这是我们不能容忍的 。解决这个问题的方案这边其实有3种：



### 方案1：poi事件模式+分页处理

看poi的文档，其实poi 的api提供了2种 模式，

- 一种就是我们常用的用户模式usermodel，就是将excel的内容一次性全部加载至 内存中，
- 还有一种用的是 事件模式，因为其实将 的excel文件后缀名.xlsx 改为 .zip，打开zip包 就可以知道其实excel的底层数据是以 xml的形式存储的。所以 事件模式就是 使用sax解析excel 。

poi提供一套api去处理 解析过程，这个api对用户不是很友好，有点晦涩难懂，我们可以通过自行封装这一套 api然后  进行分页paging 分批的去读取数据到内存中  如下所示：

#### 依赖包

```bash


        <dependency>
            <groupId>joda-time</groupId>
            <artifactId>joda-time</artifactId>
            <version>2.9.8</version>
        </dependency>
        <!-- apache poi 操作Microsoft Document -->
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi</artifactId>
            <version>3.17</version>
        </dependency>
        <!-- Apache POI - Java API To Access Microsoft Format Files-->

        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi-ooxml</artifactId>
            <version>3.17</version>
        </dependency>

        <!-- Apache POI - Java API To Access Microsoft Format Files-->
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi-ooxml-schemas</artifactId>
            <version>3.17</version>
        </dependency>
        <!-- poi eventmodel方式 依赖包 -->
        <dependency>
            <groupId>xerces</groupId>
            <artifactId>xercesImpl</artifactId>
            <version>2.11.0</version>
        </dependency>
```



#### 获取excel 的总行数

```java
package com.yinshi.gitstats.poi.sax;
import java.io.InputStream;
import java.util.regex.Pattern;

import org.apache.poi.openxml4j.opc.OPCPackage;
import org.apache.poi.xssf.eventusermodel.XSSFReader;
import org.apache.poi.xssf.model.SharedStringsTable;
import org.springframework.util.StringUtils;
import org.xml.sax.Attributes;
import org.xml.sax.ContentHandler;
import org.xml.sax.InputSource;
import org.xml.sax.SAXException;
import org.xml.sax.XMLReader;
import org.xml.sax.helpers.DefaultHandler;
import org.xml.sax.helpers.XMLReaderFactory;


/**
 * XSSF and SAX (Event API) basic example.
 * See {@link XLSX2CSV} for a fuller example of doing
 *  XSLX processing with the XSSF Event code.
 */
public class MyExcel2007ForMaxRow {
    //new add
    public long maxRow = 0;//记录总行数

    private String filename = null;
    public MyExcel2007ForMaxRow(String filename) throws Exception{
        if(StringUtils.isEmpty(filename)) {
            throw new Exception("文件名不能空");
        }
        this.filename = filename;
        processFirstSheet();
    }
    /**
     * 指定获取第一个sheet
     * @throws Exception
     */
    private void processFirstSheet() throws Exception {
        OPCPackage pkg = OPCPackage.open(filename);
        XSSFReader r = new XSSFReader( pkg );
        SharedStringsTable sst = r.getSharedStringsTable();

        XMLReader parser = fetchSheetParser(sst);

        // To look up the Sheet Name / Sheet Order / rID,
        //  you need to process the core Workbook stream.
        // Normally it's of the form rId# or rSheet#
        InputStream sheet2 = r.getSheet("rId1");
        InputSource sheetSource = new InputSource(sheet2);
        parser.parse(sheetSource);
        sheet2.close();
    }

    private XMLReader fetchSheetParser(SharedStringsTable sst) throws SAXException {
        XMLReader parser =
                XMLReaderFactory.createXMLReader(
                        "org.apache.xerces.parsers.SAXParser"
                );
        ContentHandler handler = new MaxRowHandler();
        parser.setContentHandler(handler);
        return parser;
    }

    /**
     * See org.xml.sax.helpers.DefaultHandler javadocs
     */
    private  class MaxRowHandler extends DefaultHandler {

        @Override
        public void startElement(String uri, String localName, String name,
                                 Attributes attributes) throws SAXException {
            // c => cell
            if(name.equals("c")) {
                String index = attributes.getValue("r");
                if(Pattern.compile("A[0-9]+$").matcher(index).find()){
                    maxRow++;
                }

            }

        }

    }

    public static void main(String[] args) throws Exception {
        MyExcel2007ForMaxRow reader = new MyExcel2007ForMaxRow("/home/yinshi/Documents/CodeSource/test-write07.xlsx");

        System.out.println("\n---"+reader.maxRow);

    }
}

```

#### 分页获取数据

```java
package com.yinshi.gitstats.poi.sax;


import com.yinshi.gitstats.entity.TmCase;
import org.apache.poi.openxml4j.opc.OPCPackage;
import org.apache.poi.xssf.eventusermodel.XSSFReader;
import org.apache.poi.xssf.model.SharedStringsTable;
import org.apache.poi.xssf.usermodel.XSSFRichTextString;
import org.springframework.util.StringUtils;
import org.xml.sax.*;
import org.xml.sax.helpers.DefaultHandler;
import org.xml.sax.helpers.XMLReaderFactory;

import java.io.File;
import java.io.InputStream;
import java.util.ArrayList;
import java.util.List;
import java.util.regex.Pattern;

/**
 * XSSF and SAX (Event API) basic example.
 * See {@link XLSX2CSV} for a fuller
 * example of doing XSLX processing with the XSSF Event code.
 * 目前函数又个缺陷，便是starElement函数中发现新行的策略有问题
 */
//@Service
public class MyExcel2007ForPaging_high_baK_bak<T> {

//    @Autowired
//    private TmCaseRepo tmCaseRepo;

    /**
     * 代表Excel中必须有值的起始列(A、B、C....AA、AB...)
     */
    private static final String indexC4Data = "A";

    /**
     * 存储所有行的值
     */
    public List<List<IndexValue>> dataList = new ArrayList<List<IndexValue>>();


    /*
     * 临时存储当前行的值
     */
    private List<IndexValue> rowData;

    private final int startRow;
    private final int endRow;
    private int currentRow = 0;
    private final String filename;
    /**
     * 一行文件有几个字段（列的数量）
     * 首先需要 xlsx 的excel文件的header有几个字段，建议还是手动传进来，
     * 1. 场景：防止一些没有header的文件，
     * 2. 场景： 如果一行有9列，其中第4列之后就再也没有值了，后（9-4）=5个字段，就需要手动补充空格了，否则将list转为对象vo的时候，
     * 容易数组越界异常，建议还是在这里给它补充完成 ，如果不在这里补充完整，也可以在 list转vo的时候，把list的剩余字段补上
     */
    private final int columnSize;
//    private int startRow;
//    private int endRow;
//    private int currentRow = 0;
//    private String filename;

//    //    @PostConstruct
//    private void init(String filename, int startRow, int endRow) throws Exception {
//        if (StringUtils.isEmpty(filename)) {
//            throw new Exception("文件名不能空");
//        }
//        this.filename = filename;
//        this.startRow = startRow;
//        this.endRow = endRow;
//    }


    public MyExcel2007ForPaging_high_baK_bak(String filename, int columnSize, int startRow, int endRow) throws Exception {
        if (StringUtils.isEmpty(filename)) {
            throw new Exception("文件名不能空");
        }
        this.columnSize = columnSize;
        this.filename = filename;
        this.startRow = startRow;
        this.endRow = endRow;
        processFirstSheet();

    }

    /**
     * 指定获取第一个sheet
     *
     * @throws Exception
     */
    public void processFirstSheet() throws Exception {

//        init(filename,startRow,endRow);

        OPCPackage pkg = OPCPackage.open(filename);
        XSSFReader r = new XSSFReader(pkg);
        SharedStringsTable sst = r.getSharedStringsTable();

        XMLReader parser = fetchSheetParser(sst);

        // To look up the Sheet Name / Sheet Order / rID,
        // you need to process the core Workbook stream.
        // Normally it's of the form rId# or rSheet#
        InputStream sheet1 = r.getSheet("rId1");
        InputSource sheetSource = new InputSource(sheet1);
        parser.parse(sheetSource);
        sheet1.close();
        pkg.close();
    }

    private XMLReader fetchSheetParser(SharedStringsTable sst) throws SAXException {
        XMLReader parser = XMLReaderFactory.createXMLReader("org.apache.xerces.parsers.SAXParser");
        ContentHandler handler = new PagingHandler(sst);
        parser.setContentHandler(handler);
        return parser;
    }

    /**
     * See org.xml.sax.helpers.DefaultHandler javadocs
     */
    private class PagingHandler extends DefaultHandler {
        private SharedStringsTable sst;
        private String lastContents;
        private boolean nextIsString;
        private String index = null;

        private PagingHandler(SharedStringsTable sst) {
            this.sst = sst;
        }

        /**
         * 每个单元格开始时的处理
         */
        @Override
        public void startElement(String uri, String localName, String name, Attributes attributes) throws SAXException {
            // c => cell
            if (name.equals("c")) {
                // Print the cell reference
                // System.out.print(attributes.getValue("r") + " - ");

                index = attributes.getValue("r");
                System.out.println(index);
                if (index.contains("N")) {
                    System.out.println("##" + attributes + "##");
                }

                // 这是一个新行
                if (Pattern.compile("^" + indexC4Data + "[0-9]+$").matcher(index).find()) {

                    // 存储上一行数据
                    if (rowData != null && isAccess() && !rowData.isEmpty()) {
                        dataList.add(rowData);
                    }
                    rowData = new ArrayList<IndexValue>();
                    // 新行要先清除上一行的数据
                    currentRow++;// 当前行+1
                    // System.out.println(currentRow);
                }
                if (isAccess()) {
                    // Figure out if the value is an index in the SST
                    String cellType = attributes.getValue("t");
                    if (cellType != null && cellType.equals("s")) {
                        nextIsString = true;
                    } else {
                        nextIsString = false;
                    }
                }

            }
            // Clear contents cache
            lastContents = "";
        }

        /**
         * 每个单元格结束时的处理
         */
        @Override
        public void endElement(String uri, String localName, String name) throws SAXException {
            if (isAccess()) {
                // Process the last contents as required.
                // Do now, as characters() may be called more than once
                if (nextIsString) {
                    int idx = Integer.parseInt(lastContents);
                    lastContents = new XSSFRichTextString(sst.getEntryAt(idx)).toString();
                    nextIsString = false;
                }

                // v => contents of a cell
                // Output after we've seen the string contents
                if (name.equals("v")) {
                    System.out.println(lastContents);

                    rowData.add(new IndexValue(index, lastContents));

                }
            }

        }

        /**
         * 目前流的方式值支持  Excel单元格是文本  格式；日期、数字、公式不支持
         */
        @Override
        public void characters(char[] ch, int start, int length) throws SAXException {
            if (isAccess()) {
                lastContents += new String(ch, start, length);
            }

        }

        /**
         * 如果文档结束后，发现读取的末尾行正处在当前行中，存储下这行
         * （存在这样一种情况，当待读取的末尾行正好是文档最后一行时，最后一行无法存到集合中，
         * 因为最后一行没有下一行了，所以不为启动starElement()方法， 当然我们可以通过指定最大列来处理，但不想那么做，扩展性不好）
         */
        @Override
        public void endDocument() throws SAXException {
            if (rowData != null && isAccess() && !rowData.isEmpty()) {
                dataList.add(rowData);
                System.out.println("--end");
            }

        }

    }

    private boolean isAccess() {
        if (currentRow >= startRow && currentRow <= endRow) {
            return true;
        }
        return false;
    }

    private class IndexValue {
        String v_index;
        String v_value;

        public IndexValue(String v_index, String v_value) {
            super();
            this.v_index = v_index;
            this.v_value = v_value;
        }

        @Override
        public String toString() {
            return "IndexValue [v_index=" + v_index + ", v_value=" + v_value + "]";
        }

        /**
         * 去掉数字部分（行信息），直接比较英文部分（列信息），计算前后两个值相距多少空列
         *
         * @param p
         * @return
         */
        public int getLevel(IndexValue p) {

			/*char[] other = p.v_index.replaceAll("[0-9]", "").toCharArray();
			char[] self = this.v_index.replaceAll("[0-9]", "").toCharArray();
			if (other.length != self.length)
				return -1;
			for (int i = 0; i < other.length; i++) {
				if (i == other.length - 1) {
					return self[i] - other[i];
				} else {
					if (self[i] != other[i]) {
						return -1;
					}
				}
			}
			return -1;*/

            String other = p.v_index.replaceAll("[0-9]", "");
            String self = this.v_index.replaceAll("[0-9]", "");
            return MathUtil.fromNumberSystem26(self) - MathUtil.fromNumberSystem26(other);

        }
    }

    /**
     * 获取真实的数据（处理空格，空行）
     *
     * @throws Exception
     */
    public List<T> getMyDataList(ExcelResultHandler<T> handler) throws Exception {
        // 1.首先需要 xlsx 的excel文件的header有几个字段，建议还是手动传进来，防止一些没有header的文件，以及

        int MAX_LIE = 0;
        List<T> myDataList = new ArrayList<T>();
        if (dataList == null || dataList.size() <= 0) {
            return myDataList;
        }
        /*
         * 是否是最后一行的数据
         */
        boolean islastRow = false;
        for (int i = 0; i < dataList.size(); i++) {
            List<IndexValue> i_list = dataList.get(i);
            List<String> row = new ArrayList<String>();
            TmCase tmCase = new TmCase();
            int j = 0;
            for (; j < i_list.size() - 1; j++) {
//            for (; j < 10 - 1; j++) {
                // 获取当前值,并存储
                IndexValue current = i_list.get(j);
                //去掉空格
                String tempV = current.v_value != null ? current.v_value.trim() : current.v_value;
                row.add(tempV);
                // 预存下一个
                IndexValue next = i_list.get(j + 1);
                // 获取差值
                int level = next.getLevel(current);
                System.out.println("level:==>" + level);
				/*if(i==2214){
					System.out.println("--"+i);
				}*/
                if (level <= 0) {
                    System.err.println("---!!!到达最后一行，行号：" + (i + 1) + ";level:" + level + "[超出处理范围]");
                    islastRow = true;
                    break;
                }
                //将差值补充为null，
                for (int k = 0; k < level - 1; k++) {
                    // 空值处理
                    row.add("");
                }
            }
            /*
             * 每行的最后一个值，留在最后插入
             * 但最后一行除外
             */
            if (!islastRow) {
                row.add(i_list.get(j).v_value);
            }
            // 如果row.size不等于 列数，比如总共9列，可说第5列之后就没有 数据了，那么 i_list.size=5,需要额外补充4个
            if (row.size() < this.columnSize) {
                int remain = this.columnSize - row.size();
                System.out.println("还差===》" + remain);
                for (int k = 0; k < remain; k++) {
                    row.add("");
                }
            }
            T item = handler.handler(row);
            myDataList.add(item);

        }
        return myDataList;
    }

    public static void main(String[] args) throws Exception {
        File file = new File("/home/yinshi/Documents/CodeSource/test-write09.xlsx");
//        MyExcel2007ForPaging_high myExcel2007ForPaging_high = new MyExcel2007ForPaging_high();
//        myExcel2007ForPaging_high.processFirstSheet(file.getPath(), 2, 50001);
        List<TmCase> myDataList = new MyExcel2007ForPaging_high_baK_bak(file.getPath(), 10, 2, 50001).
                getMyDataList(new TmCaseExcelResultHandler());
        for (TmCase tmCase : myDataList) {
            System.out.println(tmCase.toString());
        }
//        System.out.println(myDataList);
    }
}
```



#### 数据行转对象Handler

```java
package com.yinshi.gitstats.poi.sax;

import java.util.List;

@FunctionalInterface
public interface ExcelResultHandler<T> {

    T handler(List<String> sourceList);
}
//////////////////////////////////////////////////////////////
package com.yinshi.gitstats.poi.sax;

import com.yinshi.gitstats.entity.TmCase;

import java.nio.charset.StandardCharsets;
import java.util.List;

public class TmCaseExcelResultHandler implements ExcelResultHandler<TmCase>{
    @Override
    public TmCase handler(List<String> sourceList) {
        return transList2TmCase(sourceList);
    }
    private TmCase transList2TmCase(List<String> sourceData) {
        System.out.println(sourceData.toString());
        TmCase tmCase = new TmCase();
        tmCase.setLoanNo(sourceData.get(0));
        tmCase.setColDateStr(sourceData.get(1));
        tmCase.setColType(sourceData.get(2));
        tmCase.setColPerson(sourceData.get(3));
        tmCase.setColAddress(sourceData.get(4));
        tmCase.setColStatus(sourceData.get(5));
        tmCase.setConcreteColContent(sourceData.get(6));
        tmCase.setRepayDateStr(sourceData.get(7));
        tmCase.setRepayAmtStr(sourceData.get(8));
        tmCase.setErrorMsg(sourceData.get(9));
        return tmCase;
    }

    public static void main(String[] args) {
        String remark="我叫音十黎明";
        System.out.println(remark.length());
        System.out.println(remark.getBytes(StandardCharsets.UTF_8));
    }
}

```

#### 辅助类

```java
package com.yinshi.gitstats.poi.sax;
import org.springframework.util.StringUtils;
public class MathUtil {

    /**
     * /// <summary>
     * /// 将指定的自然数转换为26进制表示。映射关系：[1-26] ->[A-Z]。
     * /// </summary>
     * /// <param name="n">自然数（如果无效，则返回空字符串）。</param>
     * /// <returns>26进制表示。</returns>
     */
    public static String toNumberSystem26(int n) {
        String s = "";
        while (n > 0) {
            int m = n % 26;
            if (m == 0) {
                m = 26;
            }
            s = (char) (m + 64) + s;
            n = (n - m) / 26;
        }
        return s;
    }

    /**
     * <summary>
     * 将指定的26进制表示转换为自然数。映射关系：[A-Z] ->[1-26]。
     * </summary>
     * <param name="s">26进制表示（如果无效，则返回0）。</param>
     * <returns>自然数。</returns>
     */
    public static int fromNumberSystem26(String s) {
        if (StringUtils.isEmpty(s)) {
            return 0;
        }
        s = s.toUpperCase();
        int n = 0;
        char[] arr = s.toCharArray();
        for (int i = arr.length - 1, j = 1; i >= 0; i--, j *= 26) {
            char c = arr[i];
            if (c < 'A' || c > 'Z') {
                return 0;
            }
            //A的ASCII值为65
            n += ((int) c - 64) * j;
        }
        return n;
    }


    public static void main(String[] args) {
        System.out.println(fromNumberSystem26("aa"));
        System.out.println(toNumberSystem26(27));
    }
}
```

#### 对外提供服务类

```java
package com.yinshi.gitstats.poi.sax;

import com.yinshi.gitstats.entity.TmCase;

import java.io.File;
import java.util.LinkedList;
import java.util.List;

public class ReaderSaxPOIUtils<T> {
//    private static ReaderSaxPOIUtils instance = null;
    /**
     * 头文件
     */
    public final int TITLELINE_ROW_INDEX = 0;
    private ExcelResultHandler<T> excelResultHandler;

    public ReaderSaxPOIUtils(ExcelResultHandler<T> excelResultHandler) {
        this.excelResultHandler = excelResultHandler;
    }

//    public static ReaderSaxPOIUtils getInstance() {
//        if (instance == null) {
//            instance = new ReaderSaxPOIUtils();
//        }
//        return instance;
//    }


    /**
     * 获取文件标题
     * <p>
     * 一般 文件的header就是第一行，所以
     *
     * @param file            源文件
     * @param sumOfHeaderRows 代表标题header 占用几行，通常情况下 header就是第一行，sunofRow设置为 1 就可以
     * @return
     * @throws Exception
     */
//    public List<List<String>> getTitles(File file, int sumOfHeaderRows) throws Exception {
//    public List<List<String>> getTitles(File file, int sumOfHeaderRows) throws Exception {
//        //由于这里标题就是具体内容，所以应该把标题当做要获取的数据
//        return this.getPagingData(file, 1, sumOfHeaderRows, sumOfHeaderRows, 0);
//    }

    /**
     * 利用POI的sax方式分页读取2007的Excel
     *
     * @param file
     * @param pageIndex      页号 从1开始
     * @param pagesizeRows   页内行数
     * @param totalCountRows 实际的总行数（去除标题）
     * @param outOfTitleRow  忽略标题行的行数，负数忽略，0代表包含标题，正值代表跳过的标题数
     * @return
     * @throws Exception
     */
    public List<T> getPagingData(File file, int pageIndex,
                                            int pagesizeRows, int totalCountRows, int outOfTitleRow) throws Exception {

        // 总记录行数（除去标题）
        int sumOfRows = totalCountRows;

        // SAX解析方式第一行是1，不是0
        int headLineRowNum = this.TITLELINE_ROW_INDEX;
        if (outOfTitleRow > 0) {
            headLineRowNum += outOfTitleRow;
        }
        // 循环的默认起始和结束
        int startRow = headLineRowNum + 1;
        int endRow = startRow + pagesizeRows - 1;
        // 最后一行的行号
        int lastRowNum = sumOfRows + headLineRowNum;
        int odd = sumOfRows - ((pageIndex - 1) * pagesizeRows);// 本页之前的所有记录
        if ((pageIndex - 1) * pagesizeRows >= sumOfRows) {// 本页之前的所有记录已经超过总数了
            return new LinkedList<T>();
        } else if (odd > 0 && odd <= pagesizeRows) {// 剩下的行数不够本页显示数

            startRow = (headLineRowNum + 1) + (pageIndex - 1) * pagesizeRows;
            endRow = lastRowNum;
        } else {// 所剩行数充足
            startRow = headLineRowNum + (pageIndex - 1) * pagesizeRows + 1;
            endRow = headLineRowNum + pageIndex * pagesizeRows;
        }

        return new MyExcel2007ForPaging_high(file.getPath(), 10, startRow, endRow).getMyDataList(excelResultHandler);
    }


    /**
     * 采用SAX读取2007版的xlsx形式的Excel的最大行
     *
     * @param f
     * @return
     * @throws Exception
     */
    public long getSize4TheFile(File f) throws Exception {

        MyExcel2007ForMaxRow reader = new MyExcel2007ForMaxRow(f.getPath());
        return reader.maxRow - (this.TITLELINE_ROW_INDEX + 1);

    }

    public static void main(String[] args) throws Exception {
        File file = new File("/home/yinshi/Documents/CodeSource/test-write08.xlsx");
        System.out.println(file.getPath());
        System.out.println(file.getName());
//        TmCaseExcelResultHandler excelResultHandler = new TmCaseExcelResultHandler();
//        ReaderSaxPOIUtils<TmCase> saxPOIUtils = new ReaderSaxPOIUtils<>(excelResultHandler);
//        long size4TheFile = saxPOIUtils.getSize4TheFile(file);
//        System.out.println(size4TheFile);
////        List<List<String>> titles = saxPOIUtils.getTitles(file, 2);
////        System.out.println("===>" + titles);
//
//        List<TmCase> pagingData = saxPOIUtils.getPagingData(file, 1, 10, (int) size4TheFile, 1);
//        System.out.println(pagingData);
//        System.out.println("======>" + pagingData.size());
        String fileNma="222.xlsx";
        String replace = fileNma.replace(".xlsx", "");
        System.out.println(replace);
    }


}
```



#### 总结：

1. getDataList本来 返回的是一个 List<List<String>>类型的结果集，因为一个单元格Cell解析出来的就是一个string字符串，所以一行rowData 代表的是List<String>，那一页sheet 代表的就是 List<List<String>> ,
2. 我本来想使用策略模式，类似与JDBC的ResultHandler 一样，自行处理得到的结果集 ，就是 ExcelResultHandler 做的事情，将得到的rowData 转换为一个 泛型类， 但是感觉这样又有一个弊端， 就是处理 头文件 headerTitle 的数据的时候有些问题，因为头文件本身就不是一个 对象，它就应该是一个 List<String> 的 集合，所以我把  getTitle 方法获取 headerTitle的方法 给注释了 ，总之 这一块需要特殊处理
3.  再说一下使用方式 ，我们已经获取到了 文件的总行数totaCount ，我们就可以 进行分页了，就像 mybatis分页一样，我们可以自定义pageSize 一页有多少数据，然后使用pageCount= totalCount / pageSize，去计算页数，再在自己的 业务入口类中去 for（int pageIndex=1; pageIndex++;pageIndex<=pageSize）循环获取分页数据进行处理 
4. 最后说一下， 插入数据库 的方法建议自己重写一下，不要一条一条的insert， 太慢了，自己定义一个commitCount，比如2000条提交一次 



### 方案2：流式处理  

| origin | https://github.com/monitorjbl/excel-streaming-reader.git |
| ------ | -------------------------------------------------------- |
|        |                                                          |

#### 依赖包

```bash


        <!-- https://mvnrepository.com/artifact/com.monitorjbl/xlsx-streamer -->
        <dependency>
            <groupId>com.monitorjbl</groupId>
            <artifactId>xlsx-streamer</artifactId>
            <version>2.1.0</version>
        </dependency>

        <dependency>
            <groupId>joda-time</groupId>
            <artifactId>joda-time</artifactId>
            <version>2.9.8</version>
        </dependency>
        <!-- apache poi 操作Microsoft Document -->
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi</artifactId>
            <version>3.17</version>
        </dependency>
        <!-- Apache POI - Java API To Access Microsoft Format Files-->

        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi-ooxml</artifactId>
            <version>3.17</version>
        </dependency>

        <!-- Apache POI - Java API To Access Microsoft Format Files-->
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi-ooxml-schemas</artifactId>
            <version>3.17</version>
        </dependency>
        <!-- poi eventmodel方式 依赖包 -->
        <dependency>
            <groupId>xerces</groupId>
            <artifactId>xercesImpl</artifactId>
            <version>2.11.0</version>
        </dependency>
```



#### 使用方式：

我们可以下载 这个项目的源代码看看,看里面的测试类，也可以大概知道使用方式 StreamingReaderTest

src/test/java/com/monitorjbl/xlsx/StreamingReaderTest.java

```java
package com.monitorjbl.xlsx;

import com.monitorjbl.xlsx.exceptions.MissingSheetException;
import org.apache.poi.openxml4j.opc.OPCPackage;
import org.apache.poi.openxml4j.opc.PackageAccess;
import org.apache.poi.ss.usermodel.Cell;
import org.apache.poi.ss.usermodel.CellType;
import org.apache.poi.ss.usermodel.DateUtil;
import org.apache.poi.ss.usermodel.Row;
import org.apache.poi.ss.usermodel.Sheet;
import org.apache.poi.ss.usermodel.Workbook;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import java.io.File;
import java.io.FileInputStream;
import java.io.InputStream;
import java.text.SimpleDateFormat;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Calendar;
import java.util.Date;
import java.util.GregorianCalendar;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.Spliterator;
import java.util.Spliterators;
import java.util.stream.Collectors;
import java.util.stream.StreamSupport;

import static org.apache.poi.ss.usermodel.CellType.BOOLEAN;
import static org.apache.poi.ss.usermodel.CellType.NUMERIC;
import static org.apache.poi.ss.usermodel.CellType.STRING;
import static org.apache.poi.ss.usermodel.Row.MissingCellPolicy.CREATE_NULL_AS_BLANK;
import static org.apache.poi.ss.usermodel.Row.MissingCellPolicy.RETURN_BLANK_AS_NULL;
import static org.hamcrest.CoreMatchers.equalTo;
import static org.hamcrest.CoreMatchers.nullValue;
import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.core.Is.is;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertNull;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;
import static org.junit.jupiter.api.Assertions.fail;

public class StreamingReaderTest {
  @BeforeAll
  public static void init() {
    Locale.setDefault(Locale.ENGLISH);
  }


  @Test
  public void testGetDateCellValue() throws Exception {
    try(
        InputStream is = new FileInputStream("src/test/resources/data_types.xlsx");
        Workbook wb = StreamingReader.builder().open(is);
    ) {

      List<List<Cell>> obj = new ArrayList<>();

      for(Row r : wb.getSheetAt(0)) {
        List<Cell> o = new ArrayList<>();
        for(Cell c : r) {
          o.add(c);
        }
        obj.add(o);
      }

      Date dt = obj.get(4).get(1).getDateCellValue();
      assertNotNull(dt);
      final GregorianCalendar cal = new GregorianCalendar();
      cal.setTime(dt);
      assertEquals(cal.get(Calendar.YEAR), 2014);

      // Verify LocalDateTime version is correct as well
      LocalDateTime localDateTime = obj.get(4).get(1).getLocalDateTimeCellValue();
      assertEquals(2014, localDateTime.getYear());

      try {
        obj.get(0).get(0).getDateCellValue();
        fail("Should have thrown IllegalStateException");
      } catch(IllegalStateException e) { }
    }
  }

  @Test
  public void testGetDateCellValue1904() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/1904Dates.xlsx"));
        Workbook wb = StreamingReader.builder().open(is);
    ) {

      List<List<Cell>> obj = new ArrayList<>();

      for(Row r : wb.getSheetAt(0)) {
        List<Cell> o = new ArrayList<>();
        for(Cell c : r) {
          o.add(c);
        }
        obj.add(o);
      }

      Date dt = obj.get(1).get(5).getDateCellValue();
      assertNotNull(dt);
      final GregorianCalendar cal = new GregorianCalendar();
      cal.setTime(dt);
      assertEquals(cal.get(Calendar.YEAR), 1991);

      try {
        obj.get(0).get(0).getDateCellValue();
        fail("Should have thrown IllegalStateException");
      } catch(IllegalStateException e) { }
    }
  }

  @Test
  public void testGetFirstCellNum() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/gaps.xlsx"));
        Workbook wb = StreamingReader.builder().open(is);
    ) {

      List<List<Cell>> obj = new ArrayList<>();
      List<Row> rows = new ArrayList<>();
      for(Row r : wb.getSheetAt(0)) {
        rows.add(r);
        List<Cell> o = new ArrayList<>();
        for(Cell c : r) {
          o.add(c);
        }
        obj.add(o);
      }

      assertEquals(3, rows.size());
      assertEquals(3, rows.get(2).getFirstCellNum());
    }
  }

  @Test
  public void testGaps() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/gaps.xlsx"));
        Workbook wb = StreamingReader.builder().open(is);
    ) {
      List<List<Cell>> obj = new ArrayList<>();

      for(Row r : wb.getSheetAt(0)) {
        List<Cell> o = new ArrayList<>();
        for(Cell c : r) {
          o.add(c);
        }
        obj.add(o);
      }

      assertEquals(3, obj.size());
      List<Cell> row;

      row = obj.get(0);
      assertEquals(2, row.size());
      assertEquals(STRING, row.get(0).getCellType());
      assertEquals(STRING, row.get(1).getCellType());
      assertEquals("Dat", row.get(0).getStringCellValue());
      assertEquals("Dat", row.get(0).getRichStringCellValue().getString());
      assertEquals(0, row.get(0).getColumnIndex());
      assertEquals(0, row.get(0).getRowIndex());
      assertEquals("gap", row.get(1).getStringCellValue());
      assertEquals("gap", row.get(1).getRichStringCellValue().getString());
      assertEquals(2, row.get(1).getColumnIndex());
      assertEquals(0, row.get(1).getRowIndex());

      row = obj.get(1);
      assertEquals(2, row.size());
      assertEquals(STRING, row.get(0).getCellType());
      assertEquals(STRING, row.get(1).getCellType());
      assertEquals("guuurrrrrl", row.get(0).getStringCellValue());
      assertEquals("guuurrrrrl", row.get(0).getRichStringCellValue().getString());
      assertEquals(0, row.get(0).getColumnIndex());
      assertEquals(6, row.get(0).getRowIndex());
      assertEquals("!", row.get(1).getStringCellValue());
      assertEquals("!", row.get(1).getRichStringCellValue().getString());
      assertEquals(6, row.get(1).getColumnIndex());
      assertEquals(6, row.get(1).getRowIndex());
    }
  }

  @Test
  public void testMultipleSheets_alpha() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/sheets.xlsx"));
        Workbook wb = StreamingReader.builder().open(is);
    ) {
      List<List<Cell>> obj = new ArrayList<>();

      for(Row r : wb.getSheetAt(0)) {
        List<Cell> o = new ArrayList<>();
        for(Cell c : r) {
          o.add(c);
        }
        obj.add(o);
      }

      assertEquals(1, obj.size());
      List<Cell> row;

      row = obj.get(0);
      assertEquals(1, row.size());
      assertEquals("stuff", row.get(0).getStringCellValue());
      assertEquals("stuff", row.get(0).getRichStringCellValue().getString());
    }
  }

  @Test
  public void testMultipleSheets_zulu() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/sheets.xlsx"));
        Workbook wb = StreamingReader.builder().open(is);
    ) {

      List<List<Cell>> obj = new ArrayList<>();

      for(Row r : wb.getSheetAt(1)) {
        List<Cell> o = new ArrayList<>();
        for(Cell c : r) {
          o.add(c);
        }
        obj.add(o);
      }

      assertEquals(1, obj.size());
      List<Cell> row;

      row = obj.get(0);
      assertEquals(1, row.size());
      assertEquals("yeah", row.get(0).getStringCellValue());
      assertEquals("yeah", row.get(0).getRichStringCellValue().getString());
    }
  }

  @Test
  public void testSheetName_zulu() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/sheets.xlsx"));
        Workbook wb = StreamingReader.builder().open(is);
    ) {

      List<List<Cell>> obj = new ArrayList<>();

      for(Row r : wb.getSheet("SheetZulu")) {
        List<Cell> o = new ArrayList<>();
        for(Cell c : r) {
          o.add(c);
        }
        obj.add(o);
      }

      assertEquals(1, obj.size());
      List<Cell> row;

      row = obj.get(0);
      assertEquals(1, row.size());
      assertEquals("yeah", row.get(0).getStringCellValue());
      assertEquals("yeah", row.get(0).getRichStringCellValue().getString());
    }
  }

  @Test
  public void testSheetName_alpha() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/sheets.xlsx"));
        Workbook wb = StreamingReader.builder().open(is);
    ) {
      List<List<Cell>> obj = new ArrayList<>();

      for(Row r : wb.getSheet("SheetAlpha")) {
        List<Cell> o = new ArrayList<>();
        for(Cell c : r) {
          o.add(c);
        }
        obj.add(o);
      }

      assertEquals(1, obj.size());
      List<Cell> row;

      row = obj.get(0);
      assertEquals(1, row.size());
      assertEquals("stuff", row.get(0).getStringCellValue());
      assertEquals("stuff", row.get(0).getRichStringCellValue().getString());
    }
  }

  @Test
  public void testSheetName_missingInStream() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/sheets.xlsx"));
        Workbook wb = StreamingReader.builder().open(is);
    ) {
      assertThrows(MissingSheetException.class, ()->wb.getSheet("asdfasdfasdf"));
    }
  }

  @Test
  public void testSheetName_missingInFile() throws Exception {
    File f = new File("src/test/resources/sheets.xlsx");
    try(Workbook wb = StreamingReader.builder().open(f)) {
      wb.getSheet("asdfasdfasdf");
      fail("Should have failed");
    } catch(MissingSheetException e) {
      assertTrue(f.exists());
    }
  }

  @Test
  public void testIteration() throws Exception {
    File f = new File("src/test/resources/large.xlsx");
    try(
        Workbook wb = StreamingReader.builder()
            .rowCacheSize(5)
            .open(f)) {
      int i = 1;
      for(Row r : wb.getSheetAt(0)) {
        assertEquals(i, r.getCell(0).getNumericCellValue(), 0);
        assertEquals("#" + i, r.getCell(1).getStringCellValue());
        assertEquals("#" + i, r.getCell(1).getRichStringCellValue().getString());
        i++;
      }
    }
  }

  @Test
  public void testLeadingZeroes() throws Exception {
    File f = new File("src/test/resources/leadingZeroes.xlsx");

    try(Workbook wb = StreamingReader.builder().open(f)) {
      Iterator<Row> iter = wb.getSheetAt(0).iterator();
      iter.hasNext();

      Row r1 = iter.next();
      assertEquals(1, r1.getCell(0).getNumericCellValue(), 0);
      assertEquals("1", r1.getCell(0).getStringCellValue());
      assertEquals(NUMERIC, r1.getCell(0).getCellType());

      Row r2 = iter.next();
      assertEquals(2, r2.getCell(0).getNumericCellValue(), 0);
      assertEquals("0002", r2.getCell(0).getStringCellValue());
      assertEquals("0002", r2.getCell(0).getRichStringCellValue().getString());
      assertEquals(STRING, r2.getCell(0).getCellType());
    }
  }

  @Test
  public void testReadingEmptyFile() throws Exception {
    File f = new File("src/test/resources/empty_sheet.xlsx");

    try(Workbook wb = StreamingReader.builder().open(f)) {
      Iterator<Row> iter = wb.getSheetAt(0).iterator();
      assertThat(iter.hasNext(), is(false));
    }
  }

  @Test
  public void testSpecialStyles() throws Exception {
    File f = new File("src/test/resources/special_types.xlsx");

    Map<Integer, List<Cell>> contents = new HashMap<>();
    try(Workbook wb = StreamingReader.builder().open(f)) {
      for(Row row : wb.getSheetAt(0)) {
        contents.put(row.getRowNum(), new ArrayList<Cell>());
        for(Cell c : row) {
          if(c.getColumnIndex() > 0) {
            contents.get(row.getRowNum()).add(c);
          }
        }
      }
    }

    SimpleDateFormat df = new SimpleDateFormat("dd/MM/yyyy");

    assertThat(contents.size(), equalTo(2));
    assertThat(contents.get(0).size(), equalTo(4));
    assertThat(contents.get(0).get(0).getStringCellValue(), equalTo("Thu\", \"Dec 25\", \"14"));
    assertThat(contents.get(0).get(0).getDateCellValue(), equalTo(df.parse("25/12/2014")));
    assertThat(contents.get(0).get(1).getStringCellValue(), equalTo("02/04/15"));
    assertThat(contents.get(0).get(1).getDateCellValue(), equalTo(df.parse("04/02/2015")));
    assertThat(contents.get(0).get(2).getStringCellValue(), equalTo("14\". \"Mar\". \"2015"));
    assertThat(contents.get(0).get(2).getDateCellValue(), equalTo(df.parse("14/03/2015")));
    assertThat(contents.get(0).get(3).getStringCellValue(), equalTo("2015-05-05"));
    assertThat(contents.get(0).get(3).getDateCellValue(), equalTo(df.parse("05/05/2015")));

    assertThat(contents.get(1).size(), equalTo(4));
    assertThat(contents.get(1).get(0).getStringCellValue(), equalTo("3.12"));
    assertThat(contents.get(1).get(0).getNumericCellValue(), equalTo(3.12312312312));
    assertThat(contents.get(1).get(1).getStringCellValue(), equalTo("1,023,042"));
    assertThat(contents.get(1).get(1).getNumericCellValue(), equalTo(1023042.0));
    assertThat(contents.get(1).get(2).getStringCellValue(), equalTo("-312,231.12"));
    assertThat(contents.get(1).get(2).getNumericCellValue(), equalTo(-312231.12123145));
    assertThat(contents.get(1).get(3).getStringCellValue(), equalTo("(132)"));
    assertThat(contents.get(1).get(3).getNumericCellValue(), equalTo(-132.0));
  }

  @Test
  public void testBlankNumerics() throws Exception {
    File f = new File("src/test/resources/blank_cells.xlsx");
    try(Workbook wb = StreamingReader.builder().open(f)) {
      Row row = wb.getSheetAt(0).iterator().next();
      assertThat(row.getCell(1).getStringCellValue(), equalTo(""));
      assertThat(row.getCell(1).getRichStringCellValue().getString(), equalTo(""));
      assertThat(row.getCell(1).getDateCellValue(), is(nullValue()));
      assertThat(row.getCell(1).getNumericCellValue(), equalTo(0.0));
    }
  }

  @Test
  public void testFirstRowNumIs0() throws Exception {
    File f = new File("src/test/resources/data_types.xlsx");
    try(Workbook wb = StreamingReader.builder().open(f)) {
      Row row = wb.getSheetAt(0).iterator().next();
      assertThat(row.getRowNum(), equalTo(0));
    }
  }

  @Test
  public void testNoTypeCell() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/no_type_cell.xlsx"));
        Workbook wb = StreamingReader.builder().open(is)) {
      for(Row r : wb.getSheetAt(0)) {
        for(Cell c : r) {
          assertEquals("1", c.getStringCellValue());
        }
      }
    }
  }

  @Test
  public void testEncryption() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/encrypted.xlsx"));
        Workbook wb = StreamingReader.builder().password("test").open(is)) {
      OUTER:
      for(Row r : wb.getSheetAt(0)) {
        for(Cell c : r) {
          assertEquals("Demo", c.getStringCellValue());
          assertEquals("Demo", c.getRichStringCellValue().getString());
          break OUTER;
        }
      }
    }
  }

  @Test
  public void testStringCellValue() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/blank_cell_StringCellValue.xlsx"));
        Workbook wb = StreamingReader.builder().open(is);
    ) {
      for(Row r : wb.getSheetAt(0)) {
        if(r.getRowNum() == 1) {
          assertEquals("", r.getCell(1).getStringCellValue());
          assertEquals("", r.getCell(1).getRichStringCellValue().getString());
        }
      }
    }
  }

  @Test
  public void testNullValueType() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/null_celltype.xlsx"));
        Workbook wb = StreamingReader.builder().open(is);
    ) {
      for(Row r : wb.getSheetAt(0)) {
        for(Cell cell : r) {
          if(r.getRowNum() == 0 && cell.getColumnIndex() == 8) {
            assertEquals(NUMERIC, cell.getCellType());
            assertEquals("8:00:00", cell.getStringCellValue());
          }
        }
      }
    }
  }

  @Test
  public void testInlineCells() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/inline.xlsx"));
        Workbook wb = StreamingReader.builder().open(is);
    ) {
      Row row = wb.getSheetAt(0).iterator().next();
      assertEquals("First inline cell", row.getCell(0).getStringCellValue());
      assertEquals("First inline cell", row.getCell(0).getRichStringCellValue().getString());
      assertEquals("Second inline cell", row.getCell(1).getStringCellValue());
      assertEquals("Second inline cell", row.getCell(1).getRichStringCellValue().getString());
    }
  }

  @Test
  public void testMissingRattrs() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/missing-r-attrs.xlsx"));
        StreamingReader reader = StreamingReader.builder().read(is);
    ) {
      Row row = reader.iterator().next();
      assertEquals(0, row.getRowNum());
      assertEquals("1", row.getCell(0).getStringCellValue());
      assertEquals("5", row.getCell(4).getStringCellValue());
      row = reader.iterator().next();
      assertEquals(1, row.getRowNum());
      assertEquals("6", row.getCell(0).getStringCellValue());
      assertEquals("10", row.getCell(4).getStringCellValue());
      row = reader.iterator().next();
      assertEquals(6, row.getRowNum());
      assertEquals("11", row.getCell(0).getStringCellValue());
      assertEquals("15", row.getCell(4).getStringCellValue());

      assertFalse(reader.iterator().hasNext());
    }
  }

  @Test
  public void testClosingFiles() throws Exception {
    OPCPackage o = OPCPackage.open(new File("src/test/resources/blank_cell_StringCellValue.xlsx"), PackageAccess.READ);
    o.close();
  }

  @Test
  public void shouldIgnoreSpreadsheetDrawingRows() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/has_spreadsheetdrawing.xlsx"));
        Workbook wb = StreamingReader.builder().open(is);
    ) {
      Iterator<Row> iterator = wb.getSheetAt(0).iterator();
      while(iterator.hasNext()) {
        iterator.next();
      }
    }
  }

  @Test
  public void testShouldReturnNullForMissingCellPolicy_RETURN_BLANK_AS_NULL() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/blank_cells.xlsx"));
        Workbook wb = StreamingReader.builder().open(is);
    ) {
      Row row = wb.getSheetAt(0).iterator().next();
      assertNotNull(row.getCell(0, RETURN_BLANK_AS_NULL)); //Remain unchanged
      assertNull(row.getCell(1, RETURN_BLANK_AS_NULL));
    }
  }

  @Test
  public void testShouldReturnBlankForMissingCellPolicy_CREATE_NULL_AS_BLANK() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/null_cell.xlsx"));
        Workbook wb = StreamingReader.builder().open(is);
    ) {
      Row row = wb.getSheetAt(0).iterator().next();
      assertEquals("B1 is Null ->", row.getCell(0, CREATE_NULL_AS_BLANK).getStringCellValue()); //Remain unchanged
      assertEquals("B1 is Null ->", row.getCell(0, CREATE_NULL_AS_BLANK).getRichStringCellValue().getString()); //Remain unchanged
      assertThat(row.getCell(1), is(nullValue()));
      assertNotNull(row.getCell(1, CREATE_NULL_AS_BLANK));
    }
  }


  // Handle a file with a blank SST reference, like <c r="L42" s="1" t="s"><v></v></c>
  // Normally, if Excel saves the file, that whole <c ...></c> wouldn't even be there.
  @Test
  public void testShouldHandleBlankSSTReference() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/blank_sst_reference_doctored.xlsx"));
        Workbook wb = StreamingReader.builder().open(is);
    ) {
      Iterator<Row> iterator = wb.getSheetAt(0).iterator();
      while(iterator.hasNext()) {
        iterator.next();
      }
    }
  }

  // The last cell on this sheet should be a NUMERIC but there is a lingering "f"
  // tag that was getting attached to the last cell causing it to be a FORUMLA.
  @Test
  public void testForumulaOutsideCellIgnored() throws Exception {
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/formula_outside_cell.xlsx"));
        Workbook wb = StreamingReader.builder().open(is);
    ) {
      Iterator<Row> rows = wb.getSheetAt(0).iterator();
      Cell cell = null;
      while(rows.hasNext()) {
        Iterator<Cell> cells = rows.next().iterator();
        while(cells.hasNext()) {
            cell = cells.next();
        }
      }
      assertNotNull(cell);
      assertThat(cell.getCellType(), is(CellType.NUMERIC));
    }
  }

  @Test
  public void testFormulaWithDifferentTypes() throws Exception {
    try(
      InputStream is = new FileInputStream(new File("src/test/resources/formula_test.xlsx"));
      Workbook wb = StreamingReader.builder().open(is)
    ) {
      Sheet sheet = wb.getSheetAt(0);
      Iterator<Row> rowIterator = sheet.rowIterator();

      Row next = rowIterator.next();
      Cell cell = next.getCell(0);

      assertThat(cell.getCellType(), is(CellType.STRING));

      next = rowIterator.next();
      cell = next.getCell(0);

      assertThat(cell.getCellType(), is(CellType.FORMULA));
      assertThat(cell.getCachedFormulaResultType(), is(CellType.STRING));

      next = rowIterator.next();
      cell = next.getCell(0);

      assertThat(cell.getCellType(), is(CellType.FORMULA));
      assertThat(cell.getCachedFormulaResultType(), is(CellType.BOOLEAN));

      next = rowIterator.next();
      cell = next.getCell(0);

      assertThat(cell.getCellType(), is(CellType.FORMULA));
      assertThat(cell.getCachedFormulaResultType(), is(CellType.NUMERIC));
    }
  }
  
  @Test
  public void testShouldIncrementColumnNumberIfExplicitCellAddressMissing() throws Exception {
	// On consecutive columns the <c> element might miss an "r" attribute, which indicate the cell position.
	// This might be an optimization triggered by file size and specific to a particular excel version.
	// The excel would read such a file without complaining.
    try(
        InputStream is = new FileInputStream(new File("src/test/resources/sparse-columns.xlsx"));
        Workbook wb = StreamingReader.builder().open(is);
    ) {
    	 Sheet sheet = wb.getSheetAt(0);
    	 
    	 Iterator<Row> rowIterator = sheet.rowIterator();
         Row row = rowIterator.next();
         
         assertThat(row.getCell(0).getStringCellValue(), is("sparse"));
         assertThat(row.getCell(3).getStringCellValue(), is("columns"));
         assertThat(row.getCell(4).getNumericCellValue(), is(0.0));
         assertThat(row.getCell(5).getNumericCellValue(), is(1.0));

    }
  }
}

```

#### 总结：

因为我也没有仔细研究， 因为没有采用这个方式，但是 看了一下，应该不是 很难

### 方案3： 阿里easyExcel 

- 最后说一下 使用阿里的easyExcel 的背景，我本来使用第一个方案 实现了 大文件的解析， 可是晚上接到电话，说 行内代码扫描说我的实现方式 存在安全漏洞，需要去修改 ，我震惊了，

- 但是还是想 了解一下原因,出现问题，分析问题，再解决问题嘛，解决不了咱在换个方案嘛，
- 但是又害怕 扭不到 行内的安全部门，所以得两手准备，一方面定位安全问题，一方面准备新方案 （ali easyexcel）



#### 分析：

因为也是第一次使用easyExcel，再简单 看了下 说明文档后，大致知道我应该怎么用， 跟我想象中的使用方法还是有差别的 。可以说是开辟了一个新思路吧 

- 我最开始本以为 api也会提供类似与分页的方式 ，来让用户取到数据，然后自行处理，后面发现并不是  ，以读操作为例，
- 我们需要继承 AnalysisEventListener 类，实现抽象方法，在抽象方法中写 自己的业务逻辑 。
- read方法大部分的方法都是 void类型的 ，业务的逻辑都放在了 AnalysisEventListener 中去做，只有一个方法 

以下的方法是： api中能为我们提供返回值 的方法，其实原理就是在自定义的listener（SyncReadListener）中新增一个类变量List<resultVO>，然后提供get 和set方法 ，每一条rowData都会经过 invoke 方法，每次处理成功之后，就将invoke处理过后的数据，加入 list，最后 我们就可以拿到 返回值了， 

- 其实按照类似的思路，我们也可以自己实现一个带有分页功能的 Listener ，然后对外提供 分页能力，思路大概是： 
- - 新建builder 继承  AbstractExcelReaderParameterBuilder 新增分页方法， 
  - 新建类 factory 继承 EasyExcelFactory ，新增分页方法

但是感觉好像确实没必要，毕竟我们拿到返回的list<excelVO>，我们还是需要 逐条解析逐条处理。 我们的业务 处理逻辑完全可以写在  listener 里面提供的2个方法  invoke 和 doAfterAllAnalysed 里面 。

** 嘿嘿，没事给自己找事，下次写一个玩玩吧 （TODO tocheck）**



```java
    /**
     * Quickly read small files，no more than 10,000 lines.
     *
     * @param in
     *            the POI filesystem that contains the Workbook stream.
     * @param sheet
     *            read sheet.
     * @return analysis result.
     * @deprecated please use 'EasyExcel.read(in).sheet(sheetNo).doReadSync();'
     */
    @Deprecated
    public static List<Object> read(InputStream in, Sheet sheet) {
        final List<Object> rows = new ArrayList<Object>();
        new ExcelReader(in, null, new AnalysisEventListener<Object>() {
            @Override
            public void invoke(Object object, AnalysisContext context) {
                rows.add(object);
            }

            @Override
            public void doAfterAllAnalysed(AnalysisContext context) {}
        }, false).read(sheet);
        return rows;
    }


public class ExcelReaderSheetBuilder extends AbstractExcelReaderParameterBuilder<ExcelReaderSheetBuilder, ReadSheet> {
    
/**
     * Synchronous reads return results
     *
     * @return
     */
    public <T> List<T> doReadSync() {
        if (excelReader == null) {
            throw new ExcelAnalysisException("Must use 'EasyExcelFactory.read().sheet()' to call this method");
        }
        SyncReadListener syncReadListener = new SyncReadListener();
        registerReadListener(syncReadListener);
        excelReader.read(build());
        excelReader.finish();
        return (List<T>)syncReadListener.getList();
    }
}



/**
 * Synchronous data reading
 *
 * @author Jiaju Zhuang
 */
public class SyncReadListener extends AnalysisEventListener<Object> {
    private List<Object> list = new ArrayList<Object>();

    @Override
    public void invoke(Object object, AnalysisContext context) {
        list.add(object);
    }

    @Override
    public void doAfterAllAnalysed(AnalysisContext context) {}

    public List<Object> getList() {
        return list;
    }

    public void setList(List<Object> list) {
        this.list = list;
    }
}
```



#### 需求：

讲了一堆，进入正题吧，我的业务逻辑  有4点：

1. 需要校验头文件 headerTItle 的正确性，如果 valid不通过，需要直接抛出异常，不进行后续解析操作 
2. 需要校验每条数据的 正确性
3. 需要将正确的数据 批量插入数据库中
4. 需要将错误数据写在 ErrorData写在 excel 里面

#### 实现：

1. 关于校验头文件，业务需求使得我们 需要单独 新建一个 HeaderTitleValidListener，在  单纯的只是 checkheader，不做其他操作,校验不通过就抛出异常，中断解析操作
2.  关于每条数据的正确性，我们需要再新增一个 XXDataListener ，然后我们可以自行封装 一个 业务逻辑的 XXvalidtor ，方法就是 checkData，返回值就是 错误信息，这个需要写入excel的
3.  正确数据的处理，放在 listener的 invoke 方法中，新增一个类变量 greenDataCacheList，用于存储正确的数据，定义一个常量 COMMIT_COUNT=2000，当 greenDataCacheList 的size= COMMIT_COUNT ，我们就执行一个插入数据库的工作，然后clear greenDataCacheList的数据
4. 将错误数据写在 ErrorData写在 excel 里面，这个其实可以放在 doAfterAllAnalysed里面去做，当然也可以同样的方式放在 invoke 方法里面去做 

思路清晰了，实现起来就很简单了 



#### 依赖包

```bash
<!-- https://mvnrepository.com/artifact/com.alibaba/easyexcel -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>easyexcel</artifactId>
            <version>2.2.6</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/com.monitorjbl/xlsx-streamer -->
        <dependency>
            <groupId>com.monitorjbl</groupId>
            <artifactId>xlsx-streamer</artifactId>
            <version>2.1.0</version>
        </dependency>

        <dependency>
            <groupId>joda-time</groupId>
            <artifactId>joda-time</artifactId>
            <version>2.9.8</version>
        </dependency>
        <!-- apache poi 操作Microsoft Document -->
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi</artifactId>
            <version>3.17</version>
        </dependency>
        <!-- Apache POI - Java API To Access Microsoft Format Files-->

        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi-ooxml</artifactId>
            <version>3.17</version>
        </dependency>

        <!-- Apache POI - Java API To Access Microsoft Format Files-->
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi-ooxml-schemas</artifactId>
            <version>3.17</version>
        </dependency>
        <!-- poi eventmodel方式 依赖包 -->
        <dependency>
            <groupId>xerces</groupId>
            <artifactId>xercesImpl</artifactId>
            <version>2.11.0</version>
        </dependency>
```



#### 自定义LIstener

```java
package com.yinshi.gitstats.poi.sax.easyexcel;

import java.util.Arrays;
import java.util.List;
import java.util.Map;

import com.alibaba.excel.exception.ExcelCommonException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.alibaba.excel.context.AnalysisContext;
import com.alibaba.excel.event.AnalysisEventListener;
import com.alibaba.fastjson.JSON;


public class HeaderTitleValidListener extends AnalysisEventListener<TmCase> {
    private static final Logger LOGGER = LoggerFactory.getLogger(HeaderTitleValidListener.class);
    /**
     * 每隔5条存储数据库，实际使用中可以3000条，然后清理list ，方便内存回收
     */
    private static final int BATCH_COUNT = 5;

    @Override
    public void invokeHeadMap(Map<Integer, String> headMap, AnalysisContext context) {
        if (!isValidHeadertitle(headMap)) {
            throw new ExcelCommonException("头文件异常");
        }
        LOGGER.info("HEAD:{}", JSON.toJSONString(headMap));
        LOGGER.info("total:{}", context.readSheetHolder().getTotal());

    }

    /**
     * true： 通过
     * false：  未通过
     *
     * @param headMap
     * @return
     */
    private boolean isValidHeadertitle(Map<Integer, String> headMap) {

        final String[] ORGIN_HEADERTITLE = {"姓名", "年龄"};
        List<String> ORGIN_HEADERTITLE_List = Arrays.asList(ORGIN_HEADERTITLE);
        //TODO 校验逻辑
        // 1. 校验excel的头文件是否为空
        // 2. 校验excel的头文件 的长度 headMap.entrySet().size  ORGIN_HEADERTITLE_List.size
        // 3. 校验excel的头文件 every item

        return true;

    }

    /**
     * 每条shuju
     * @param data
     *            one row value. Is is same as {@link AnalysisContext#readRowHolder()}
     * @param context
     */
    @Override
    public void invoke(TmCase data, AnalysisContext context) {

        // donothing
//        LOGGER.info("index:{}", context.readRowHolder().getRowIndex());
//        LOGGER.info("解析到一条数据:{}", JSON.toJSONString(data));
    }

    @Override
    public void doAfterAllAnalysed(AnalysisContext context) {
        // donothing
//        LOGGER.info("所有数据解析完成！");
    }

}


/////////////////////////////////////////////////////////////////////////////////////////////////////
///// 分界线
/////////////////////////////////////////////////////////////////////////////////////////////
package com.yinshi.gitstats.poi.sax.easyexcel;

import com.alibaba.excel.context.AnalysisContext;
import com.alibaba.excel.event.AnalysisEventListener;
import com.alibaba.fastjson.JSON;
import com.yinshi.gitstats.entity.TmCase;
import org.junit.Assert;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;


public class LargeDataListener extends AnalysisEventListener<TmCase> {
    private static final Logger LOGGER = LoggerFactory.getLogger(LargeDataListener.class);
    private int count = 0;
	/**
     * 每隔5条存储数据库，实际使用中可以3000条，然后清理list ，方便内存回收
     */
    private static final int BATCH_COUNT = 2000;
    private List<TmCase> greenDataList = new ArrayList<>(2000);


    @Override
    public void invoke(TmCase data, AnalysisContext context) {
        // 1. 如果 greenDataList大小==BATCH_COUNT ，则插入数据库，然后 greenDataList.clear
        // 2. 执行数据校验Validtor，校验通过，greenDataList.add
        // 3. Validtor校验不通过，则将 errorMsg ,set到对象中
        
        
        
    }

    @Override
    public void doAfterAllAnalysed(AnalysisContext context) {
        LOGGER.info("Large row count:{}", count);
        // 将错误数据写入 excel
    }
}

///////////////////////////////////////////////////////////////////////////////////////
///////////////////////////////////////////////////////////////////////////////////////
package com.yinshi.gitstats.entity;

import com.alibaba.excel.annotation.ExcelIgnore;
import com.alibaba.excel.annotation.ExcelProperty;
import lombok.Data;
import lombok.ToString;

import javax.persistence.*;

@Entity
@ToString
@Data
@Table(name = "tm_case")
public class TmCase {
    /**
     * 序列化ID
     */
    private static final long serialVersionUID = -4112999956266472980L;
//
    @ExcelIgnore
    @Id
    @Column(name = "id", columnDefinition = "BIGINT")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private String id;

    //    @ExcelProperty(index = 2, converter = GenderConverter.class)
    @ExcelProperty(index = 0)
    @Column(name = "loanNo")
    private String loanNo;

    @ExcelProperty(index = 1)
    @Column(name = "colDateStr")
    private String colDateStr;

    @ExcelProperty(index = 2)
    @Column(name = "colType")
    private String colType;

    @ExcelProperty(index = 3)
    @Column(name = "colPerson")
    private String colPerson;

    @ExcelProperty(index = 4)
    @Column(name = "colAddress")
    private String colAddress;

    @ExcelProperty(index = 5)
    @Column(name = "colStatus")
    private String colStatus;

    @ExcelProperty(index = 6)
    @Column(name = "concreteColContent")
    private String concreteColContent;

    @ExcelProperty(index = 7)
    @Column(name = "repayDateStr")
    private String repayDateStr;

    @ExcelProperty(index = 8)
    @Column(name = "repayAmtStr")
    private String repayAmtStr;

    @ExcelProperty(index = 9)
    @Column(name = "errorMsg")
    private String errorMsg;


}

```



#### 总结： 

- 写的都是伪代码，思路提供了，自行实现吧 ，代码都在公司上，自己电脑只是验证可行性 

- 可以多看看 easyExcel的 源码，还是很nice的 

  | origin | https://github.com/alibaba/easyexcel.git |
  | ------ | ---------------------------------------- |
  |        |                                          |





### 总结

最终通过实验，我们比较一下这写方案的性能差异：

```java
  public static void main(String[] args) throws Exception {
      //参数自行改动
 // VM:  -Xmx40M -Xms40M -XX:+PrintGCDetails -XX:+PrintHeapAtGC
//        readXlsxTest();
//        testSAX();
        testEasyExcel();

    }
    public static void testEasyExcel(){
        File file = new File("/home/yinshi/Documents/CodeSource/test-write071.xlsx");
        long start = System.currentTimeMillis();
        EasyExcel.read(file, TmCase.class,
                new LargeDataListener()).headRowNumber(2).sheet().doRead();
        System.out.println("Large data total time spent:"+ (System.currentTimeMillis() - start));
    }

    public static void testSAX() throws Exception {
        File file = new File("/home/yinshi/Documents/CodeSource/test-write071.xlsx");
//
//        List<TmCase> myDataList = new MyExcel2007ForPaging_high_baK_bak(file.getPath(), 10, 2, 50001).
//                getMyDataList(new TmCaseExcelResultHandler());
        ReaderSaxPOIUtils<TmCase> utils = new ReaderSaxPOIUtils<TmCase>(new TmCaseExcelResultHandler());
        List<TmCase> pagingData = utils.getPagingData(file, 1, 2000, 50001, 1);
        for (TmCase tmCase : pagingData) {
            System.out.println(tmCase.toString());
        }
    }

    /**
     * 读取以 .xlsx 为后缀的 Excel 测试
     *
     * @throws IOException 文件操作异常
     */
    public static void readXlsxTest() throws IOException, IOException {
        // 读取 Excel 文件
        // 获取文件路径和文件
        File file = new File("/home/yinshi/Documents/CodeSource/test-write071.xlsx");

        FileInputStream fis = new FileInputStream(file);
        // 将输入流转换为工作簿对象
        XSSFWorkbook workbook = new XSSFWorkbook(fis);
        // 获取第一个工作表
//        XSSFSheet sheet = workbook.getSheet("sheet0");
        // 使用索引获取工作表
        XSSFSheet sheet = workbook.getSheetAt(0);
        // 获取指定行
        XSSFRow row = sheet.getRow(1);
        // 获取指定列
        XSSFCell cell = row.getCell(2);
        // 打印
        System.out.println(cell.getStringCellValue());
    }
```



![](/uploads/poiJVM/excelSize.png)

![](/uploads/poiJVM/userModelJVM.png)

![](/uploads/poiJVM/SaxPOI_JVMparams.png)

![](/uploads/poiJVM/ali_easyExcel_JVM.png)

计算一下，excel 大小为 2.1 M

用户模式需要： 新生代used + 老年代used = (23327+30563 )/1024= 52.6M

sax事件驱动模式： （9408+31764 ） /1024=  40.2 M

阿里easyExcel：（8699+15950）/1024=24.07 M

所以..................厉害



**这边再贴一下关于 streamReader  和 sax的解析xml 的安全问题的 源头，以及解决方案**

ref : https://www.dazhuanlan.com/2020/02/02/5e36a1acefeea/