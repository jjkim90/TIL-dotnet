# WinForm으로 만드는 사용자 인터페이스

## WinForm 이벤트 기반 프로그래밍

### 이벤트 기반 프로그래밍의 이해
```csharp
// 전통적인 콘솔 프로그램 (절차적)
Console.WriteLine("이름을 입력하세요:");
string name = Console.ReadLine();
Console.WriteLine($"안녕하세요, {name}님!");

// WinForm (이벤트 기반)
public partial class MainForm : Form
{
    private TextBox nameTextBox;
    private Button greetButton;
    
    public MainForm()
    {
        InitializeComponent();
        
        // 이벤트 핸들러 등록
        greetButton.Click += GreetButton_Click;
    }
    
    private void GreetButton_Click(object sender, EventArgs e)
    {
        MessageBox.Show($"안녕하세요, {nameTextBox.Text}님!");
    }
}
```

## C# 코드로 WinForm 윈도우 만들기

### 기본 Form 생성
```csharp
using System;
using System.Windows.Forms;

public class SimpleForm : Form
{
    public SimpleForm()
    {
        // Form 속성 설정
        this.Text = "My First WinForm";
        this.Width = 400;
        this.Height = 300;
        this.StartPosition = FormStartPosition.CenterScreen;
        
        // 버튼 추가
        Button button = new Button();
        button.Text = "Click Me!";
        button.Location = new Point(150, 100);
        button.Size = new Size(100, 30);
        button.Click += Button_Click;
        
        // Form에 컨트롤 추가
        this.Controls.Add(button);
    }
    
    private void Button_Click(object sender, EventArgs e)
    {
        MessageBox.Show("버튼이 클릭되었습니다!");
    }
    
    [STAThread]
    static void Main()
    {
        Application.EnableVisualStyles();
        Application.SetCompatibleTextRenderingDefault(false);
        Application.Run(new SimpleForm());
    }
}
```

## Application 클래스

### Application 기본 메서드
```csharp
public class Program
{
    [STAThread]
    static void Main()
    {
        // 비주얼 스타일 활성화
        Application.EnableVisualStyles();
        Application.SetCompatibleTextRenderingDefault(false);
        
        // 애플리케이션 정보
        Console.WriteLine($"제품명: {Application.ProductName}");
        Console.WriteLine($"버전: {Application.ProductVersion}");
        Console.WriteLine($"실행 경로: {Application.ExecutablePath}");
        
        // 애플리케이션 실행
        Application.Run(new MainForm());
        
        // Application.Run() 이후 코드는 폼이 닫힌 후 실행됨
        Console.WriteLine("애플리케이션 종료");
    }
}
```

### 메시지 필터링
```csharp
public class MessageFilter : IMessageFilter
{
    public bool PreFilterMessage(ref Message m)
    {
        // WM_KEYDOWN = 0x0100
        if (m.Msg == 0x0100)
        {
            Keys key = (Keys)m.WParam.ToInt32();
            Console.WriteLine($"키 눌림: {key}");
            
            // ESC 키는 처리하지 않음
            if (key == Keys.Escape)
                return true;  // 메시지 처리됨
        }
        
        return false;  // 메시지 계속 전달
    }
}

// 메시지 필터 등록
MessageFilter filter = new MessageFilter();
Application.AddMessageFilter(filter);
```

## Form 클래스

### Form 이벤트
```csharp
public class EventForm : Form
{
    public EventForm()
    {
        // Form 이벤트 등록
        this.Load += Form_Load;
        this.Shown += Form_Shown;
        this.FormClosing += Form_FormClosing;
        this.FormClosed += Form_FormClosed;
        this.Resize += Form_Resize;
        this.Paint += Form_Paint;
    }
    
    private void Form_Load(object sender, EventArgs e)
    {
        Console.WriteLine("Form Load - 폼이 로드됨");
    }
    
    private void Form_Shown(object sender, EventArgs e)
    {
        Console.WriteLine("Form Shown - 폼이 처음 표시됨");
    }
    
    private void Form_FormClosing(object sender, FormClosingEventArgs e)
    {
        DialogResult result = MessageBox.Show(
            "정말 종료하시겠습니까?", 
            "확인", 
            MessageBoxButtons.YesNo);
            
        if (result == DialogResult.No)
            e.Cancel = true;  // 종료 취소
    }
    
    private void Form_Paint(object sender, PaintEventArgs e)
    {
        // 그리기 작업
        e.Graphics.DrawString("Hello WinForm!", 
            new Font("Arial", 20), 
            Brushes.Black, 
            10, 10);
    }
}
```

### Form 속성 조절
```csharp
public class CustomForm : Form
{
    public CustomForm()
    {
        // 크기와 위치
        this.Size = new Size(800, 600);
        this.MinimumSize = new Size(400, 300);
        this.MaximumSize = new Size(1200, 900);
        this.StartPosition = FormStartPosition.CenterScreen;
        
        // 창 스타일
        this.FormBorderStyle = FormBorderStyle.Sizable;
        this.MaximizeBox = true;
        this.MinimizeBox = true;
        this.ControlBox = true;
        
        // 투명도
        this.Opacity = 0.95;  // 95% 불투명
        
        // 아이콘
        this.Icon = new Icon("app.ico");
        
        // 배경
        this.BackColor = Color.LightBlue;
        this.BackgroundImage = Image.FromFile("background.jpg");
        this.BackgroundImageLayout = ImageLayout.Stretch;
    }
}
```

### Form에 컨트롤 올리기
```csharp
public class ControlForm : Form
{
    private TableLayoutPanel mainLayout;
    
    public ControlForm()
    {
        InitializeComponents();
    }
    
    private void InitializeComponents()
    {
        // TableLayoutPanel로 레이아웃 구성
        mainLayout = new TableLayoutPanel();
        mainLayout.Dock = DockStyle.Fill;
        mainLayout.RowCount = 3;
        mainLayout.ColumnCount = 2;
        
        // Label
        Label nameLabel = new Label();
        nameLabel.Text = "이름:";
        nameLabel.TextAlign = ContentAlignment.MiddleRight;
        
        // TextBox
        TextBox nameTextBox = new TextBox();
        nameTextBox.Dock = DockStyle.Fill;
        
        // Button
        Button submitButton = new Button();
        submitButton.Text = "제출";
        submitButton.Click += (s, e) => 
        {
            MessageBox.Show($"입력된 이름: {nameTextBox.Text}");
        };
        
        // 레이아웃에 추가
        mainLayout.Controls.Add(nameLabel, 0, 0);
        mainLayout.Controls.Add(nameTextBox, 1, 0);
        mainLayout.Controls.Add(submitButton, 1, 1);
        
        this.Controls.Add(mainLayout);
    }
}
```

## WinForm 컨트롤들

### GroupBox와 Panel
```csharp
// GroupBox - 제목이 있는 그룹
GroupBox optionsGroup = new GroupBox();
optionsGroup.Text = "옵션 선택";
optionsGroup.Location = new Point(10, 10);
optionsGroup.Size = new Size(200, 100);

RadioButton option1 = new RadioButton();
option1.Text = "옵션 1";
option1.Location = new Point(10, 20);

RadioButton option2 = new RadioButton();
option2.Text = "옵션 2";
option2.Location = new Point(10, 40);

optionsGroup.Controls.Add(option1);
optionsGroup.Controls.Add(option2);

// Panel - 스크롤 가능한 컨테이너
Panel scrollPanel = new Panel();
scrollPanel.AutoScroll = true;
scrollPanel.BorderStyle = BorderStyle.FixedSingle;
scrollPanel.Size = new Size(300, 200);
```

### Label과 TextBox
```csharp
// Label
Label infoLabel = new Label();
infoLabel.Text = "정보 표시";
infoLabel.Font = new Font("맑은 고딕", 12, FontStyle.Bold);
infoLabel.ForeColor = Color.Blue;
infoLabel.AutoSize = true;

// TextBox
TextBox inputBox = new TextBox();
inputBox.Multiline = true;
inputBox.ScrollBars = ScrollBars.Vertical;
inputBox.MaxLength = 1000;
inputBox.PasswordChar = '*';  // 비밀번호 입력용

// 이벤트 처리
inputBox.TextChanged += (s, e) =>
{
    infoLabel.Text = $"입력된 글자 수: {inputBox.Text.Length}";
};
```

### ComboBox와 ListBox
```csharp
// ComboBox
ComboBox combo = new ComboBox();
combo.Items.AddRange(new string[] { "항목1", "항목2", "항목3" });
combo.DropDownStyle = ComboBoxStyle.DropDownList;
combo.SelectedIndexChanged += (s, e) =>
{
    MessageBox.Show($"선택: {combo.SelectedItem}");
};

// ListBox
ListBox listBox = new ListBox();
listBox.SelectionMode = SelectionMode.MultiExtended;
listBox.Items.AddRange(new object[] { "Apple", "Banana", "Cherry" });
listBox.SelectedIndexChanged += (s, e) =>
{
    var selected = listBox.SelectedItems;
    Console.WriteLine($"선택된 항목 수: {selected.Count}");
};
```

### CheckBox와 RadioButton
```csharp
// CheckBox
CheckBox agreeCheck = new CheckBox();
agreeCheck.Text = "약관에 동의합니다";
agreeCheck.CheckedChanged += (s, e) =>
{
    submitButton.Enabled = agreeCheck.Checked;
};

// RadioButton 그룹
RadioButton[] sizeOptions = new RadioButton[]
{
    new RadioButton { Text = "Small", Tag = "S" },
    new RadioButton { Text = "Medium", Tag = "M" },
    new RadioButton { Text = "Large", Tag = "L" }
};

foreach (var radio in sizeOptions)
{
    radio.CheckedChanged += (s, e) =>
    {
        if (radio.Checked)
            Console.WriteLine($"선택된 사이즈: {radio.Tag}");
    };
}
```

### TrackBar와 ProgressBar
```csharp
// TrackBar
TrackBar volumeBar = new TrackBar();
volumeBar.Minimum = 0;
volumeBar.Maximum = 100;
volumeBar.Value = 50;
volumeBar.TickFrequency = 10;
volumeBar.ValueChanged += (s, e) =>
{
    volumeLabel.Text = $"볼륨: {volumeBar.Value}%";
};

// ProgressBar
ProgressBar progress = new ProgressBar();
progress.Minimum = 0;
progress.Maximum = 100;
progress.Style = ProgressBarStyle.Continuous;

// 진행률 업데이트
async Task UpdateProgress()
{
    for (int i = 0; i <= 100; i++)
    {
        progress.Value = i;
        await Task.Delay(50);
    }
}
```

### TreeView와 ListView
```csharp
// TreeView
TreeView treeView = new TreeView();
TreeNode rootNode = new TreeNode("루트");
TreeNode childNode1 = new TreeNode("자식1");
TreeNode childNode2 = new TreeNode("자식2");

rootNode.Nodes.Add(childNode1);
rootNode.Nodes.Add(childNode2);
treeView.Nodes.Add(rootNode);

treeView.AfterSelect += (s, e) =>
{
    Console.WriteLine($"선택: {e.Node.Text}");
};

// ListView
ListView listView = new ListView();
listView.View = View.Details;
listView.Columns.Add("이름", 100);
listView.Columns.Add("크기", 80);
listView.Columns.Add("수정일", 150);

ListViewItem item1 = new ListViewItem("파일1.txt");
item1.SubItems.Add("1.5 KB");
item1.SubItems.Add("2024-01-01");
listView.Items.Add(item1);
```

## 대화상자 (Dialog)

### MessageBox
```csharp
// 기본 메시지박스
MessageBox.Show("안녕하세요!");

// 다양한 옵션
DialogResult result = MessageBox.Show(
    "파일을 저장하시겠습니까?",
    "확인",
    MessageBoxButtons.YesNoCancel,
    MessageBoxIcon.Question,
    MessageBoxDefaultButton.Button1);

switch (result)
{
    case DialogResult.Yes:
        SaveFile();
        break;
    case DialogResult.No:
        break;
    case DialogResult.Cancel:
        return;
}
```

### 파일 대화상자
```csharp
// OpenFileDialog
OpenFileDialog openDialog = new OpenFileDialog();
openDialog.Filter = "텍스트 파일 (*.txt)|*.txt|모든 파일 (*.*)|*.*";
openDialog.Title = "파일 열기";

if (openDialog.ShowDialog() == DialogResult.OK)
{
    string content = File.ReadAllText(openDialog.FileName);
    textBox.Text = content;
}

// SaveFileDialog
SaveFileDialog saveDialog = new SaveFileDialog();
saveDialog.Filter = "텍스트 파일 (*.txt)|*.txt";
saveDialog.DefaultExt = "txt";

if (saveDialog.ShowDialog() == DialogResult.OK)
{
    File.WriteAllText(saveDialog.FileName, textBox.Text);
}
```

### 색상과 폰트 대화상자
```csharp
// ColorDialog
ColorDialog colorDialog = new ColorDialog();
if (colorDialog.ShowDialog() == DialogResult.OK)
{
    this.BackColor = colorDialog.Color;
}

// FontDialog
FontDialog fontDialog = new FontDialog();
fontDialog.ShowColor = true;

if (fontDialog.ShowDialog() == DialogResult.OK)
{
    textBox.Font = fontDialog.Font;
    textBox.ForeColor = fontDialog.Color;
}
```

## 사용자 인터페이스와 비동기 작업

### UI 스레드와 작업 스레드
```csharp
public partial class AsyncForm : Form
{
    private Button startButton;
    private ProgressBar progressBar;
    private Label statusLabel;
    
    private async void StartButton_Click(object sender, EventArgs e)
    {
        startButton.Enabled = false;
        
        // Progress 객체로 진행률 보고
        var progress = new Progress<int>(percent =>
        {
            // UI 스레드에서 실행됨
            progressBar.Value = percent;
            statusLabel.Text = $"진행률: {percent}%";
        });
        
        try
        {
            await DoLongRunningTaskAsync(progress);
            MessageBox.Show("작업 완료!");
        }
        catch (Exception ex)
        {
            MessageBox.Show($"오류: {ex.Message}");
        }
        finally
        {
            startButton.Enabled = true;
        }
    }
    
    private async Task DoLongRunningTaskAsync(IProgress<int> progress)
    {
        for (int i = 0; i <= 100; i++)
        {
            // 작업 스레드에서 실행
            await Task.Run(() =>
            {
                Thread.Sleep(50);  // 작업 시뮬레이션
            });
            
            // 진행률 보고
            progress?.Report(i);
        }
    }
}
```

### BackgroundWorker (레거시)
```csharp
public class BackgroundForm : Form
{
    private BackgroundWorker worker;
    
    public BackgroundForm()
    {
        worker = new BackgroundWorker();
        worker.WorkerReportsProgress = true;
        worker.WorkerSupportsCancellation = true;
        
        worker.DoWork += Worker_DoWork;
        worker.ProgressChanged += Worker_ProgressChanged;
        worker.RunWorkerCompleted += Worker_RunWorkerCompleted;
    }
    
    private void Worker_DoWork(object sender, DoWorkEventArgs e)
    {
        for (int i = 0; i <= 100; i++)
        {
            if (worker.CancellationPending)
            {
                e.Cancel = true;
                return;
            }
            
            Thread.Sleep(50);
            worker.ReportProgress(i);
        }
    }
    
    private void Worker_ProgressChanged(object sender, ProgressChangedEventArgs e)
    {
        progressBar.Value = e.ProgressPercentage;
    }
    
    private void Worker_RunWorkerCompleted(object sender, RunWorkerCompletedEventArgs e)
    {
        if (e.Cancelled)
            MessageBox.Show("작업이 취소되었습니다.");
        else if (e.Error != null)
            MessageBox.Show($"오류: {e.Error.Message}");
        else
            MessageBox.Show("작업 완료!");
    }
}
```

## 실전 예제: 간단한 텍스트 에디터

```csharp
public class SimpleTextEditor : Form
{
    private MenuStrip menuStrip;
    private ToolStrip toolStrip;
    private TextBox textBox;
    private StatusStrip statusStrip;
    private string currentFile = null;
    
    public SimpleTextEditor()
    {
        InitializeComponents();
        UpdateTitle();
    }
    
    private void InitializeComponents()
    {
        // MenuStrip
        menuStrip = new MenuStrip();
        
        var fileMenu = new ToolStripMenuItem("파일(&F)");
        fileMenu.DropDownItems.Add("새로 만들기(&N)", null, New_Click);
        fileMenu.DropDownItems.Add("열기(&O)...", null, Open_Click);
        fileMenu.DropDownItems.Add("저장(&S)", null, Save_Click);
        fileMenu.DropDownItems.Add(new ToolStripSeparator());
        fileMenu.DropDownItems.Add("끝내기(&X)", null, Exit_Click);
        
        menuStrip.Items.Add(fileMenu);
        
        // ToolStrip
        toolStrip = new ToolStrip();
        toolStrip.Items.Add(new ToolStripButton("새로 만들기", null, New_Click));
        toolStrip.Items.Add(new ToolStripButton("열기", null, Open_Click));
        toolStrip.Items.Add(new ToolStripButton("저장", null, Save_Click));
        
        // TextBox
        textBox = new TextBox();
        textBox.Multiline = true;
        textBox.ScrollBars = ScrollBars.Both;
        textBox.Dock = DockStyle.Fill;
        textBox.Font = new Font("Consolas", 10);
        textBox.TextChanged += TextBox_TextChanged;
        
        // StatusStrip
        statusStrip = new StatusStrip();
        statusStrip.Items.Add(new ToolStripStatusLabel("준비"));
        
        // Form에 추가
        this.Controls.Add(textBox);
        this.Controls.Add(toolStrip);
        this.Controls.Add(menuStrip);
        this.Controls.Add(statusStrip);
        this.MainMenuStrip = menuStrip;
        
        this.Size = new Size(800, 600);
        this.StartPosition = FormStartPosition.CenterScreen;
    }
    
    private void New_Click(object sender, EventArgs e)
    {
        if (ConfirmSave())
        {
            textBox.Clear();
            currentFile = null;
            UpdateTitle();
        }
    }
    
    private void Open_Click(object sender, EventArgs e)
    {
        if (ConfirmSave())
        {
            OpenFileDialog dialog = new OpenFileDialog();
            dialog.Filter = "텍스트 파일 (*.txt)|*.txt|모든 파일 (*.*)|*.*";
            
            if (dialog.ShowDialog() == DialogResult.OK)
            {
                textBox.Text = File.ReadAllText(dialog.FileName);
                currentFile = dialog.FileName;
                UpdateTitle();
            }
        }
    }
    
    private void Save_Click(object sender, EventArgs e)
    {
        SaveFile();
    }
    
    private bool SaveFile()
    {
        if (currentFile == null)
        {
            SaveFileDialog dialog = new SaveFileDialog();
            dialog.Filter = "텍스트 파일 (*.txt)|*.txt";
            
            if (dialog.ShowDialog() == DialogResult.OK)
            {
                currentFile = dialog.FileName;
            }
            else
            {
                return false;
            }
        }
        
        File.WriteAllText(currentFile, textBox.Text);
        UpdateTitle();
        return true;
    }
    
    private void UpdateTitle()
    {
        string fileName = currentFile != null ? 
            Path.GetFileName(currentFile) : "제목 없음";
        this.Text = $"{fileName} - 간단한 텍스트 에디터";
    }
    
    private bool ConfirmSave()
    {
        if (textBox.Modified)
        {
            var result = MessageBox.Show(
                "변경사항을 저장하시겠습니까?",
                "확인",
                MessageBoxButtons.YesNoCancel);
                
            if (result == DialogResult.Yes)
                return SaveFile();
            else if (result == DialogResult.Cancel)
                return false;
        }
        return true;
    }
}
```

## 핵심 개념 정리
- **이벤트 기반**: 사용자 동작에 반응하는 프로그래밍
- **Form**: WinForm 애플리케이션의 기본 창
- **Control**: 사용자와 상호작용하는 UI 요소
- **Layout**: 컨트롤 배치와 크기 조절
- **비동기 UI**: async/await로 반응성 유지