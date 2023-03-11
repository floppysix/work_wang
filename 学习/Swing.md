```java
System.setProperty("java.awt.headless", "false");  
        // 创建 JFrame 实例  
        JFrame frame = new JFrame("历史数据导入-txt(无前缀）");  
        // Setting the width and height of frame  
//        frame.setSize(350, 200);  
        frame.setBounds(600, 400, 700, 400);  
        frame.setDefaultCloseOperation(WindowConstants.EXIT_ON_CLOSE);  
  
        /* 创建面板，这个类似于 HTML 的 div 标签  
         * 我们可以创建多个面板并在 JFrame 中指定位置  
         * 面板中我们可以添加文本字段，按钮及其他组件。  
         */        JPanel panel = new JPanel();  
        // 添加面板  
        frame.add(panel);  
        /*  
         * 调用用户定义的方法并添加组件到面板  
         */        /* 布局部分我们这边不多做介绍  
         * 这边设置布局为 null         */        panel.setLayout(null);  
  
        // 创建 JLabel        JLabel userLabel = new JLabel("文件夹位置：");  
        /* 这个方法定义了组件的位置。  
         * setBounds(x, y, width, height)         * x 和 y 指定左上角的新位置，由 width 和 height 指定新的大小。  
         */        userLabel.setBounds(10, 20, 80, 25);  
        panel.add(userLabel);  
  
        /*  
         * 创建文本域用于用户输入  
         */        JTextField userText = new JTextField(20);  
        userText.setBounds(100, 20, 400, 25);  
        panel.add(userText);  
  
        JLabel passwordLabel = new JLabel("导入文件名:");  
        passwordLabel.setBounds(10,50,80,25);  
        panel.add(passwordLabel);  
  
        JTextField fileText = new JTextField(20);  
        fileText.setBounds(100, 50, 400, 25);  
        panel.add(fileText);  
  
        JButton jb = new JButton("打开文件");  
        jb.setActionCommand("open");  
        jb.setBounds(500, 20, 160, 25);  
        panel.add(jb);  
        jb.addActionListener(new ActionListener() {  
  
            @Override  
            public void actionPerformed(ActionEvent e) {  
                if (e.getActionCommand().equals("open")){  
                    JFileChooser jf = new JFileChooser();  
                    jf.setFileSelectionMode(JFileChooser.FILES_AND_DIRECTORIES);  
                    jf.showOpenDialog(panel);//显示打开的文件对话框  
                    File f =  jf.getSelectedFile();//使用文件类获取选择器选择的文件  
                    String s = f.getAbsolutePath();//返回路径名  
                    userText.setText(s);  
                    //JOptionPane弹出对话框类，显示绝对路径名  
                    JOptionPane.showMessageDialog(panel, s, "导入数据文件夹",JOptionPane.WARNING_MESSAGE);  
                }  
            }  
        });  
  
        JProgressBar bar = new JProgressBar(JProgressBar.HORIZONTAL, 0, 100);  
        // 设置显示百分比  
        bar.setStringPainted(true);  
        bar.setBounds(10, 80, 490,25);  
        panel.add(bar);  
        // 创建登录按钮  
        JButton loginButton = new JButton("提交");  
        loginButton.setBounds(10, 110, 80, 25);  
        panel.add(loginButton);  
  
        loginButton.addActionListener(new ActionListener() {  
  
            @Override  
            public void actionPerformed(ActionEvent e) {  
                System.out.println(userText.getText());  
                new Thread(new Runnable() {  
  
                    @Override  
                    public void run() {  
                        saveAISController.getStart(userText.getText());  
                    }  
                }).start();  
            }  
        });  
  
        Timer timer = new Timer(200, new ActionListener() {  
            @Override  
            public void actionPerformed(ActionEvent e) {  
  
                bar.setValue(Integer.parseInt(saveAISController.per));  
                fileText.setText(saveAISController.fileName);  
  
            }  
        });  
        timer.start();  
  
        // 设置界面可见  
        frame.setVisible(true);  
    }
```