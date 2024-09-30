# 고급 컨트롤

## DataGrid

DataGrid는 테이블 형태로 데이터를 표시하고 편집할 수 있는 강력한 컨트롤입니다.

### 기본 DataGrid 사용
```xml
<DataGrid ItemsSource="{Binding People}"
          AutoGenerateColumns="True"
          CanUserAddRows="True"
          CanUserDeleteRows="True"
          CanUserSortColumns="True"
          GridLinesVisibility="All"/>
```

### 수동 컬럼 정의
```xml
<DataGrid ItemsSource="{Binding Employees}" AutoGenerateColumns="False">
    <DataGrid.Columns>
        <!-- 텍스트 컬럼 -->
        <DataGridTextColumn Header="ID" 
                            Binding="{Binding Id}" 
                            Width="50"
                            IsReadOnly="True"/>
        
        <!-- 편집 가능한 텍스트 컬럼 -->
        <DataGridTextColumn Header="Name" 
                            Binding="{Binding Name}" 
                            Width="150"/>
        
        <!-- 체크박스 컬럼 -->
        <DataGridCheckBoxColumn Header="Active" 
                                Binding="{Binding IsActive}" 
                                Width="60"/>
        
        <!-- 콤보박스 컬럼 -->
        <DataGridComboBoxColumn Header="Department"
                                SelectedItemBinding="{Binding Department}"
                                Width="120">
            <DataGridComboBoxColumn.ItemsSource>
                <x:Array Type="sys:String">
                    <sys:String>IT</sys:String>
                    <sys:String>HR</sys:String>
                    <sys:String>Sales</sys:String>
                    <sys:String>Marketing</sys:String>
                </x:Array>
            </DataGridComboBoxColumn.ItemsSource>
        </DataGridComboBoxColumn>
        
        <!-- 템플릿 컬럼 -->
        <DataGridTemplateColumn Header="Actions" Width="100">
            <DataGridTemplateColumn.CellTemplate>
                <DataTemplate>
                    <StackPanel Orientation="Horizontal">
                        <Button Content="Edit" Margin="2"
                                Command="{Binding DataContext.EditCommand, 
                                          RelativeSource={RelativeSource AncestorType=DataGrid}}"
                                CommandParameter="{Binding}"/>
                        <Button Content="Delete" Margin="2"
                                Command="{Binding DataContext.DeleteCommand, 
                                          RelativeSource={RelativeSource AncestorType=DataGrid}}"
                                CommandParameter="{Binding}"/>
                    </StackPanel>
                </DataTemplate>
            </DataGridTemplateColumn.CellTemplate>
        </DataGridTemplateColumn>
    </DataGrid.Columns>
</DataGrid>
```

### DataGrid 스타일링
```xml
<DataGrid ItemsSource="{Binding Products}">
    <DataGrid.Resources>
        <!-- 행 스타일 -->
        <Style TargetType="DataGridRow">
            <Style.Triggers>
                <Trigger Property="IsMouseOver" Value="True">
                    <Setter Property="Background" Value="LightBlue"/>
                </Trigger>
                <Trigger Property="IsSelected" Value="True">
                    <Setter Property="Background" Value="DarkBlue"/>
                    <Setter Property="Foreground" Value="White"/>
                </Trigger>
                <!-- 데이터 기반 스타일 -->
                <DataTrigger Binding="{Binding InStock}" Value="False">
                    <Setter Property="Background" Value="LightPink"/>
                </DataTrigger>
            </Style.Triggers>
        </Style>
        
        <!-- 셀 스타일 -->
        <Style TargetType="DataGridCell">
            <Style.Triggers>
                <Trigger Property="IsSelected" Value="True">
                    <Setter Property="BorderBrush" Value="Black"/>
                    <Setter Property="BorderThickness" Value="1"/>
                </Trigger>
            </Style.Triggers>
        </Style>
    </DataGrid.Resources>
    
    <!-- 교대 행 색상 -->
    <DataGrid.AlternatingRowBackground>
        <SolidColorBrush Color="#F0F0F0"/>
    </DataGrid.AlternatingRowBackground>
</DataGrid>
```

### DataGrid 그룹화
```xml
<Grid>
    <Grid.Resources>
        <CollectionViewSource x:Key="GroupedEmployees" Source="{Binding Employees}">
            <CollectionViewSource.GroupDescriptions>
                <PropertyGroupDescription PropertyName="Department"/>
            </CollectionViewSource.GroupDescriptions>
        </CollectionViewSource>
    </Grid.Resources>
    
    <DataGrid ItemsSource="{Binding Source={StaticResource GroupedEmployees}}">
        <DataGrid.GroupStyle>
            <GroupStyle>
                <GroupStyle.HeaderTemplate>
                    <DataTemplate>
                        <DockPanel Background="LightGray">
                            <TextBlock Text="{Binding Name}" FontWeight="Bold" Margin="5"/>
                            <TextBlock Text="{Binding ItemCount, StringFormat='({0} items)'}" 
                                     DockPanel.Dock="Right" Margin="5"/>
                        </DockPanel>
                    </DataTemplate>
                </GroupStyle.HeaderTemplate>
                <GroupStyle.ContainerStyle>
                    <Style TargetType="GroupItem">
                        <Setter Property="Template">
                            <Setter.Value>
                                <ControlTemplate TargetType="GroupItem">
                                    <Expander IsExpanded="True">
                                        <Expander.Header>
                                            <ContentPresenter Content="{Binding Name}"/>
                                        </Expander.Header>
                                        <ItemsPresenter/>
                                    </Expander>
                                </ControlTemplate>
                            </Setter.Value>
                        </Setter>
                    </Style>
                </GroupStyle.ContainerStyle>
            </GroupStyle>
        </DataGrid.GroupStyle>
    </DataGrid>
</Grid>
```

## TreeView

TreeView는 계층적 데이터를 표시하는 컨트롤입니다.

### HierarchicalDataTemplate 사용
```xml
<TreeView ItemsSource="{Binding RootFolders}">
    <TreeView.ItemTemplate>
        <HierarchicalDataTemplate ItemsSource="{Binding Children}">
            <StackPanel Orientation="Horizontal">
                <Image Source="{Binding Icon}" Width="16" Height="16" Margin="0,0,5,0"/>
                <TextBlock Text="{Binding Name}"/>
                <TextBlock Text="{Binding ItemCount, StringFormat=' ({0})'}" 
                         Foreground="Gray" FontSize="11"/>
            </StackPanel>
        </HierarchicalDataTemplate>
    </TreeView.ItemTemplate>
</TreeView>
```

### 다중 레벨 TreeView
```csharp
public class FileSystemItem
{
    public string Name { get; set; }
    public string FullPath { get; set; }
    public ObservableCollection<FileSystemItem> Children { get; set; }
    public bool IsDirectory { get; set; }
    public string Icon => IsDirectory ? "/Images/folder.png" : "/Images/file.png";
    
    public FileSystemItem()
    {
        Children = new ObservableCollection<FileSystemItem>();
    }
}
```

```xml
<TreeView x:Name="fileTreeView">
    <TreeView.Resources>
        <!-- 폴더 템플릿 -->
        <HierarchicalDataTemplate DataType="{x:Type local:FileSystemItem}"
                                  ItemsSource="{Binding Children}">
            <StackPanel Orientation="Horizontal">
                <Image Source="{Binding Icon}" Width="16" Height="16"/>
                <TextBlock Text="{Binding Name}" Margin="5,0,0,0"/>
            </StackPanel>
        </HierarchicalDataTemplate>
    </TreeView.Resources>
    
    <TreeView.ItemContainerStyle>
        <Style TargetType="TreeViewItem">
            <Setter Property="IsExpanded" Value="{Binding IsExpanded, Mode=TwoWay}"/>
            <Setter Property="IsSelected" Value="{Binding IsSelected, Mode=TwoWay}"/>
            <EventSetter Event="PreviewMouseRightButtonDown" 
                         Handler="TreeViewItem_PreviewMouseRightButtonDown"/>
        </Style>
    </TreeView.ItemContainerStyle>
</TreeView>
```

### TreeView 검색 및 확장
```csharp
public class TreeViewHelper
{
    public static TreeViewItem FindTreeViewItem(ItemsControl container, object item)
    {
        if (container == null || item == null) return null;
        
        if (container.DataContext == item)
        {
            return container as TreeViewItem;
        }
        
        for (int i = 0; i < container.Items.Count; i++)
        {
            var child = container.ItemContainerGenerator.ContainerFromIndex(i) as TreeViewItem;
            if (child != null)
            {
                var result = FindTreeViewItem(child, item);
                if (result != null) return result;
            }
        }
        
        return null;
    }
    
    public static void ExpandToItem(TreeView treeView, object item)
    {
        var treeViewItem = FindTreeViewItem(treeView, item);
        if (treeViewItem != null)
        {
            treeViewItem.IsExpanded = true;
            treeViewItem.BringIntoView();
            treeViewItem.IsSelected = true;
        }
    }
}
```

## TabControl

TabControl은 여러 페이지를 탭으로 구분하여 표시합니다.

### 동적 탭 생성
```xml
<TabControl ItemsSource="{Binding Documents}" 
            SelectedItem="{Binding ActiveDocument}">
    <!-- 탭 헤더 템플릿 -->
    <TabControl.ItemTemplate>
        <DataTemplate>
            <DockPanel Width="120">
                <Button DockPanel.Dock="Right" 
                        Content="×" 
                        Width="20" Height="20"
                        Background="Transparent"
                        BorderThickness="0"
                        Command="{Binding DataContext.CloseDocumentCommand, 
                                  RelativeSource={RelativeSource AncestorType=TabControl}}"
                        CommandParameter="{Binding}"/>
                <TextBlock Text="{Binding Title}" 
                         VerticalAlignment="Center"
                         TextTrimming="CharacterEllipsis"/>
            </DockPanel>
        </DataTemplate>
    </TabControl.ItemTemplate>
    
    <!-- 탭 콘텐츠 템플릿 -->
    <TabControl.ContentTemplate>
        <DataTemplate>
            <ScrollViewer>
                <TextBox Text="{Binding Content}" 
                       AcceptsReturn="True"
                       AcceptsTab="True"/>
            </ScrollViewer>
        </DataTemplate>
    </TabControl.ContentTemplate>
</TabControl>
```

### 탭 스타일링
```xml
<TabControl>
    <TabControl.Resources>
        <Style TargetType="TabItem">
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="TabItem">
                        <Border x:Name="Border" 
                                BorderBrush="Gray" 
                                BorderThickness="1,1,1,0"
                                CornerRadius="4,4,0,0"
                                Margin="2,0">
                            <ContentPresenter x:Name="ContentSite"
                                            ContentSource="Header"
                                            HorizontalAlignment="Center"
                                            VerticalAlignment="Center"
                                            Margin="10,2"/>
                        </Border>
                        <ControlTemplate.Triggers>
                            <Trigger Property="IsSelected" Value="True">
                                <Setter TargetName="Border" Property="Background" Value="White"/>
                                <Setter TargetName="Border" Property="BorderBrush" Value="Black"/>
                            </Trigger>
                            <Trigger Property="IsSelected" Value="False">
                                <Setter TargetName="Border" Property="Background" Value="LightGray"/>
                            </Trigger>
                        </ControlTemplate.Triggers>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
        </Style>
    </TabControl.Resources>
    
    <TabItem Header="General">
        <!-- Content -->
    </TabItem>
    <TabItem Header="Advanced">
        <!-- Content -->
    </TabItem>
</TabControl>
```

## Ribbon

Ribbon은 Office 스타일의 리본 인터페이스를 제공합니다.

### 기본 Ribbon 구조
```xml
<Ribbon>
    <!-- 애플리케이션 메뉴 -->
    <Ribbon.ApplicationMenu>
        <RibbonApplicationMenu SmallImageSource="/Images/app_menu.png">
            <RibbonApplicationMenuItem Header="New" 
                                      ImageSource="/Images/new.png"
                                      Command="{Binding NewCommand}"/>
            <RibbonApplicationMenuItem Header="Open" 
                                      ImageSource="/Images/open.png"
                                      Command="{Binding OpenCommand}"/>
            <RibbonApplicationMenuItem Header="Save" 
                                      ImageSource="/Images/save.png"
                                      Command="{Binding SaveCommand}"/>
            <RibbonSeparator/>
            <RibbonApplicationMenuItem Header="Exit" 
                                      ImageSource="/Images/exit.png"
                                      Command="{Binding ExitCommand}"/>
        </RibbonApplicationMenu>
    </Ribbon.ApplicationMenu>
    
    <!-- 리본 탭 -->
    <RibbonTab Header="Home">
        <RibbonGroup Header="Clipboard">
            <RibbonButton LargeImageSource="/Images/paste.png" 
                         Label="Paste" 
                         Command="{Binding PasteCommand}"/>
            <RibbonButton SmallImageSource="/Images/cut.png" 
                         Label="Cut" 
                         Command="{Binding CutCommand}"/>
            <RibbonButton SmallImageSource="/Images/copy.png" 
                         Label="Copy" 
                         Command="{Binding CopyCommand}"/>
        </RibbonGroup>
        
        <RibbonGroup Header="Font">
            <RibbonComboBox Label="Font Family" 
                           SelectedItem="{Binding SelectedFont}">
                <RibbonGallery>
                    <RibbonGalleryCategory ItemsSource="{Binding FontFamilies}"/>
                </RibbonGallery>
            </RibbonComboBox>
            
            <RibbonTextBox Label="Font Size" 
                          Text="{Binding FontSize}"/>
        </RibbonGroup>
    </RibbonTab>
    
    <RibbonTab Header="Insert">
        <RibbonGroup Header="Illustrations">
            <RibbonButton LargeImageSource="/Images/picture.png" 
                         Label="Picture" 
                         Command="{Binding InsertPictureCommand}"/>
            <RibbonButton LargeImageSource="/Images/shapes.png" 
                         Label="Shapes" 
                         Command="{Binding InsertShapeCommand}"/>
        </RibbonGroup>
    </RibbonTab>
</Ribbon>
```

## Calendar와 DatePicker

### Calendar 컨트롤
```xml
<Calendar SelectedDate="{Binding SelectedDate}"
          DisplayDateStart="{Binding MinDate}"
          DisplayDateEnd="{Binding MaxDate}"
          SelectionMode="SingleRange"
          IsTodayHighlighted="True">
    <Calendar.BlackoutDates>
        <CalendarDateRange Start="2024-12-24" End="2024-12-26"/>
        <CalendarDateRange Start="2025-01-01" End="2025-01-01"/>
    </Calendar.BlackoutDates>
</Calendar>

<!-- 이벤트 처리 -->
<Calendar SelectedDatesChanged="Calendar_SelectedDatesChanged">
    <Calendar.CalendarDayButtonStyle>
        <Style TargetType="CalendarDayButton">
            <Style.Triggers>
                <DataTrigger Binding="{Binding Path=Date.DayOfWeek}" Value="Sunday">
                    <Setter Property="Background" Value="LightPink"/>
                </DataTrigger>
                <DataTrigger Binding="{Binding Path=Date.DayOfWeek}" Value="Saturday">
                    <Setter Property="Background" Value="LightBlue"/>
                </DataTrigger>
            </Style.Triggers>
        </Style>
    </Calendar.CalendarDayButtonStyle>
</Calendar>
```

### DatePicker 커스터마이징
```xml
<DatePicker SelectedDate="{Binding BirthDate}"
            DisplayDateStart="1900-01-01"
            DisplayDateEnd="{x:Static sys:DateTime.Today}"
            FirstDayOfWeek="Monday"
            IsTodayHighlighted="True">
    <DatePicker.Resources>
        <Style TargetType="DatePickerTextBox">
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="DatePickerTextBox">
                        <Grid>
                            <Grid.ColumnDefinitions>
                                <ColumnDefinition/>
                                <ColumnDefinition Width="Auto"/>
                            </Grid.ColumnDefinitions>
                            
                            <TextBox x:Name="PART_TextBox" 
                                   Grid.Column="0"
                                   Text="{Binding Path=SelectedDate, 
                                          RelativeSource={RelativeSource AncestorType=DatePicker},
                                          StringFormat='yyyy년 MM월 dd일'}"/>
                            
                            <Button x:Name="PART_Button" 
                                   Grid.Column="1"
                                   Content="📅"
                                   Width="25"/>
                        </Grid>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
        </Style>
    </DatePicker.Resources>
</DatePicker>
```

## 핵심 개념 정리
- **DataGrid**: 테이블 형식의 데이터 표시 및 편집
- **TreeView**: 계층적 데이터 구조 표시
- **TabControl**: 다중 페이지 인터페이스
- **Ribbon**: Office 스타일 리본 메뉴
- **Calendar/DatePicker**: 날짜 선택 컨트롤
- **HierarchicalDataTemplate**: 계층적 데이터 템플릿
- **ItemContainerStyle**: 컨테이너 항목 스타일링