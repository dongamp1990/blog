---
title: Itext Pdf 使用字体
tags: [开发日志]
date: 2019-07-11
categories: [Itext]
---
## 前言
在用Itext生成pdf的时候，需要使用到中文字体，默认是不支持中文字体的，调试了很久发现需要创建一个自带的字体，自带的中文字体样式不好看，所以在此记录下这个问题。


## 代码
```
import com.itextpdf.io.font.PdfEncodings;
import com.itextpdf.kernel.font.PdfFont;
import com.itextpdf.kernel.font.PdfFontFactory;
import com.itextpdf.kernel.geom.Rectangle;
import com.itextpdf.kernel.pdf.PdfDocument;
import com.itextpdf.kernel.pdf.PdfPage;
import com.itextpdf.kernel.pdf.PdfReader;
import com.itextpdf.kernel.pdf.PdfWriter;
import com.itextpdf.kernel.pdf.canvas.PdfCanvas;
import com.itextpdf.layout.Canvas;
import com.itextpdf.layout.element.Paragraph;
import com.itextpdf.layout.element.Text;

import java.io.IOException;

public class Test {
    public static void main(String[] args) throws IOException {
        //PdfFontFactory.createFont参数解析：字体文件路径, PdfEncodings, 是否嵌入文档，
        //PdfEncodings要使用PdfEncodings.IDENTITY_H，否则无法正常显示中文
        PdfFont font = PdfFontFactory.createFont("resources/font/SourceHanSansSC-Normal.otf", PdfEncodings.IDENTITY_H, false);
        PdfWriter pdfWriter = new PdfWriter("D:\\pdf\\test.pdf");
        PdfDocument pdfDocument = new PdfDocument(pdfWriter);
        PdfPage pdfPage = pdfDocument.addNewPage();
        PdfCanvas pdfCanvas = new PdfCanvas(pdfPage);
        Canvas canvas = new Canvas(pdfCanvas, pdfDocument, new Rectangle((float)30, (float)500, (float)100, (float)40));
        canvas.add(new Paragraph(new Text("中文测试").setFont(font).setFontSize(20)));
        canvas.close();
        pdfDocument.close();
        pdfWriter.close();
    }
}


```