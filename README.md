# 231212 비주얼프로그래밍 실습코딩

20191276 컴퓨터공학과 양용석

WINUI 3 을 이용한 간단한 사진 뷰어 만들기 

1. 신규 프로젝트를 만든 뒤, Asset 폴더를 우클릭해 새 필터를 만들고, 이름을 Samples 로 한다.
2. https://github.com/microsoft/WindowsAppSDK-Samples/tree/main/Samples/PhotoEditor/cs-winui/Assets/Samples 위의 링크에 있는 이미지들을</br>
프로젝트가 있는 폴더를 들어가 Asset>Samples 폴더 안에 붙여넣고 Samples 필터를 우클릭해 기존 항목 추가를 눌러 붙여넣은 이미지들을 추가한다.
3. ImageFileInfo.idl 파일을 생성한 뒤 아래와 같은 코드를 입력한다.</br>

```
// ImageFileInfo.idl
namespace SimplePhotos
{
    runtimeclass ImageFileInfo : Microsoft.UI.Xaml.Data.INotifyPropertyChanged
    {
        ImageFileInfo(Windows.Storage.FileProperties.ImageProperties properties, Windows.Storage.StorageFile imageFile, String name, String type);
        Windows.Storage.FileProperties.ImageProperties ImageProperties{ get; };
        Windows.Storage.StorageFile ImageFile{ get; };
        String ImageName{ get; };
        String ImageFileType{ get; };
        String ImageDimensions{ get; };
        String ImageTitle;
        UInt32 ImageRating;
    }
}
```
4. 프로젝트 폴더를 들어가  \Generated Files\sources\ 경로에 있는 ImageFileInfo.h, ImageFileInfo.cpp 파일을 복사해 프로젝트 폴더에 붙여넣고 프로젝트에 포함시킨다.
   
5. pch.h 파일 끝에 다음 필수 포함을 추가한다.

```
// pch.h
...
#include <winrt/Windows.Storage.FileProperties.h>
#include <winrt/Windows.Storage.Search.h>
#include <winrt/Windows.Storage.Streams.h>
#include <winrt/Microsoft.UI.Xaml.Media.Imaging.h>
```

6. 붙여넣은 ImageFileInfo.h, ImageFileInfo.cpp 두 파일의 코드를 아래와 같이 작성한다.</br> 

```
// ImageFileInfo.h
#pragma once
#include "ImageFileInfo.g.h"

namespace winrt::SimplePhotos::implementation
{
    struct ImageFileInfo : ImageFileInfoT<ImageFileInfo>
    {
        ImageFileInfo() = default;

        ImageFileInfo(
            winrt::Windows::Storage::FileProperties::ImageProperties const& properties,
            winrt::Windows::Storage::StorageFile const& imageFile,
            hstring const& name,
            hstring const& type);

        winrt::Windows::Storage::FileProperties::ImageProperties ImageProperties()
        {
            return m_imageProperties;
        }

        winrt::Windows::Storage::StorageFile ImageFile()
        {
            return m_imageFile;
        }

        hstring ImageName()
        {
            return m_imageName;
        }

        hstring ImageFileType()
        {
            return m_imageFileType;
        }

        hstring ImageDimensions()
        {
            return to_hstring(ImageProperties().Width()) + L" x " + to_hstring(ImageProperties().Height());
        }

        hstring ImageTitle()
        {
            return ImageProperties().Title() == L"" ? ImageName() : ImageProperties().Title();
        }

        void ImageTitle(hstring const& value);

        uint32_t ImageRating()
        {
            return ImageProperties().Rating();
        }

        void ImageRating(uint32_t value);

        winrt::event_token PropertyChanged(winrt::Microsoft::UI::Xaml::Data::PropertyChangedEventHandler const& handler)
        {
            return m_propertyChanged.add(handler);
        }

        void PropertyChanged(winrt::event_token const& token) noexcept
        {
            m_propertyChanged.remove(token);
        }

        Windows::Foundation::IAsyncOperation<Microsoft::UI::Xaml::Media::Imaging::BitmapImage> GetImageSourceAsync();

        Windows::Foundation::IAsyncOperation<Microsoft::UI::Xaml::Media::Imaging::BitmapImage> GetImageThumbnailAsync();

    private:
        // File and information fields.
        Windows::Storage::FileProperties::ImageProperties m_imageProperties{ nullptr };
        Windows::Storage::StorageFile m_imageFile{ nullptr };
        hstring m_imageName;
        hstring m_imageFileType;

        event<Microsoft::UI::Xaml::Data::PropertyChangedEventHandler> m_propertyChanged;
        void OnPropertyChanged(hstring propertyName)
        {
            m_propertyChanged(*this, Microsoft::UI::Xaml::Data::PropertyChangedEventArgs(propertyName));
        }
    };
}
namespace winrt::SimplePhotos::factory_implementation
{
    struct ImageFileInfo : ImageFileInfoT<ImageFileInfo, implementation::ImageFileInfo>
    {
    };
}
```

</br>

```
// ImageFileInfo.cpp
#include "pch.h"
#include "ImageFileInfo.h"
#include "ImageFileInfo.g.cpp"
#include <random>

using namespace std;
namespace winrt
{
    using namespace Microsoft::UI::Xaml;
    using namespace Microsoft::UI::Xaml::Media::Imaging;
    using namespace Windows::Foundation;
    using namespace Windows::Storage;
    using namespace Windows::Storage::FileProperties;
    using namespace Windows::Storage::Streams;
}

namespace winrt::SimplePhotos::implementation
{
    ImageFileInfo::ImageFileInfo(
        winrt::Windows::Storage::FileProperties::ImageProperties const& properties,
        winrt::Windows::Storage::StorageFile const& imageFile,
        hstring const& name,
        hstring const& type) :
        m_imageProperties{ properties },
        m_imageName{ name },
        m_imageFileType{ type },
        m_imageFile{ imageFile }
    {
        auto rating = properties.Rating();
        auto random = std::random_device();
        std::uniform_int_distribution<> dist(1, 5);
        ImageRating(rating == 0 ? dist(random) : rating);
    }

    void ImageFileInfo::ImageTitle(hstring const& value)
    {
        if (ImageProperties().Title() != value)
        {
            ImageProperties().Title(value);
            ImageProperties().SavePropertiesAsync();
            OnPropertyChanged(L"ImageTitle");
        }
    }

    void ImageFileInfo::ImageRating(uint32_t value)
    {
        if (ImageProperties().Rating() != value)
        {
            ImageProperties().Rating(value);
            ImageProperties().SavePropertiesAsync();
            OnPropertyChanged(L"ImageRating");
        }
    }

    IAsyncOperation<BitmapImage> ImageFileInfo::GetImageSourceAsync()
    {
        IRandomAccessStream stream{ co_await ImageFile().OpenAsync(FileAccessMode::Read) };
        BitmapImage bitmap{};
        bitmap.SetSource(stream);
        co_return bitmap;
    }

    IAsyncOperation<BitmapImage> ImageFileInfo::GetImageThumbnailAsync()
    {
        auto thumbnail = co_await m_imageFile.GetThumbnailAsync(FileProperties::ThumbnailMode::PicturesView);
        BitmapImage bitmapImage{};
        bitmapImage.SetSource(thumbnail);
        thumbnail.Close();
        co_return bitmapImage;
    }
}
```
</br>

7.MainWindow.idl 파일을 아래와 같이 코드를 작성하고, MainWindow.h, cpp파일에서 MyProperty 속성을 지운 뒤 아래의 코드를 추가한다. 

```
// MainWindow.idl
namespace SimplePhotos
{
    [default_interface]
    runtimeclass MainWindow : Microsoft.UI.Xaml.Window
    {
        MainWindow();
        Windows.Foundation.Collections.IVector<IInspectable> Images{ get; };
    }
}
```

</br>

```
// MainWindow.xaml.h
...
struct MainWindow : MainWindowT<MainWindow>
{
    ...

    Windows::Foundation::Collections::IVector<Windows::Foundation::IInspectable> Images() const
    {
        return m_images;
    }

private:
    Windows::Foundation::Collections::IVector<IInspectable> m_images{ nullptr };

    Windows::Foundation::IAsyncAction GetItemsAsync();
    Windows::Foundation::IAsyncOperation<SimplePhotos::ImageFileInfo> LoadImageInfoAsync(Windows::Storage::StorageFile);
};
...
```

</br>

```
// MainWindow.xaml.cpp
...
#include "ImageFileInfo.h"
namespace winrt
{
    using namespace Windows::ApplicationModel;
    using namespace Windows::Foundation;
    using namespace Windows::Foundation::Collections;
    using namespace Windows::Storage;
    using namespace Windows::Storage::Search;
    using namespace Microsoft::UI::Xaml;
}
...
IAsyncAction MainWindow::GetItemsAsync()
{
    StorageFolder appInstalledFolder = Package::Current().InstalledLocation();
    StorageFolder picturesFolder = co_await appInstalledFolder.GetFolderAsync(L"Assets\\Samples");

    auto result = picturesFolder.CreateFileQueryWithOptions(QueryOptions());

    IVectorView<StorageFile> imageFiles = co_await result.GetFilesAsync();
    for (auto&& file : imageFiles)
    {
        Images().Append(co_await LoadImageInfoAsync(file));
    }

    ImageGridView().ItemsSource(Images());
}

IAsyncOperation<SimplePhotos::ImageFileInfo> MainWindow::LoadImageInfoAsync(StorageFile file)
{
    auto properties = co_await file.Properties().GetImagePropertiesAsync();
    SimplePhotos::ImageFileInfo info(properties,
                       file, file.DisplayName(), file.DisplayType());
    co_return info;
}
...
MainWindow::MainWindow() :
    m_images(winrt::single_threaded_observable_vector<IInspectable>())
{
    InitializeComponent();
    GetItemsAsync();
}

```

</br>


8. MainWindow.xaml 파일을 작성한다.

```
<Window
    x:Class="SimplePhotos.MainWindow"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:local="using:SimplePhotos"
    xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    mc:Ignorable="d">

    <Grid>
        <Grid.Resources>
            <DataTemplate x:Key="ImageGridView_ItemTemplate"
                           x:DataType="local:ImageFileInfo">
                <Grid Height="300"
          Width="300"
          Margin="8">
                    <Grid.RowDefinitions>
                        <RowDefinition />
                        <RowDefinition Height="Auto" />
                    </Grid.RowDefinitions>

                    <Image x:Name="ItemImage"
               Source="Assets/StoreLogo.png"
               Stretch="Uniform" />

                    <StackPanel Orientation="Vertical"
                    Grid.Row="1">
                        <TextBlock Text="{x:Bind ImageTitle}"
                       HorizontalAlignment="Center"
                       Style="{StaticResource SubtitleTextBlockStyle}" />
                        <StackPanel Orientation="Horizontal"
                        HorizontalAlignment="Center">
                            <TextBlock Text="{x:Bind ImageFileType}"
                           HorizontalAlignment="Center"
                           Style="{StaticResource CaptionTextBlockStyle}" />
                            <TextBlock Text="{x:Bind ImageDimensions}"
                           HorizontalAlignment="Center"
                           Style="{StaticResource CaptionTextBlockStyle}"
                           Margin="8,0,0,0" />
                        </StackPanel>

                        <RatingControl Value="{x:Bind ImageRating}" IsReadOnly="True"/>
                    </StackPanel>
                </Grid>
            </DataTemplate>
            <Style x:Key="ImageGridView_ItemContainerStyle"
            TargetType="GridViewItem">
                <Setter Property="Background" Value="Gray"/>
                <Setter Property="Margin" Value="8"/>
            </Style>
            <ItemsPanelTemplate x:Key="ImageGridView_ItemsPanelTemplate">
                <ItemsWrapGrid Orientation="Horizontal"
                           HorizontalAlignment="Center"/>
            </ItemsPanelTemplate>
        </Grid.Resources>
        <GridView x:Name="ImageGridView"
                   ItemContainerStyle="{StaticResource ImageGridView_ItemContainerStyle}"
                   ItemTemplate="{StaticResource ImageGridView_ItemTemplate}"
                   ItemsPanel="{StaticResource ImageGridView_ItemsPanelTemplate}"
                   ContainerContentChanging="ImageGridView_ContainerContentChanging">
        </GridView>
    </Grid>
</Window>

```

9. ContainerContentChanging 이벤트를 작성하기 위해 MainWindow.xaml.h, cpp 파일을 아래와 같이 수정한다. 이를 통해 대체 텍스트가 실제 이미지로 변경된다.

```
//MainWindow.xaml.cpp
 namespace winrt
{
    using namespace Windows::ApplicationModel;
    using namespace Windows::Foundation;
    using namespace Windows::Foundation::Collections;
    using namespace Windows::Storage;
    using namespace Windows::Storage::Search;
    using namespace Microsoft::UI::Xaml;
    using namespace Microsoft::UI::Xaml::Controls;
}

 namespace winrt::SimplePhotos::implementation
 {
     IAsyncAction MainWindow::GetItemsAsync()
    {
        StorageFolder appInstalledFolder = Package::Current().InstalledLocation();
        StorageFolder picturesFolder = co_await appInstalledFolder.GetFolderAsync(L"Assets\\Samples");

        auto result = picturesFolder.CreateFileQueryWithOptions(QueryOptions());

        IVectorView<StorageFile> imageFiles = co_await result.GetFilesAsync();
        for (auto&& file : imageFiles)
        {
            Images().Append(co_await LoadImageInfoAsync(file));
        }

        ImageGridView().ItemsSource(Images());
    }

    IAsyncOperation<SimplePhotos::ImageFileInfo> MainWindow::LoadImageInfoAsync(StorageFile file)
    {
        auto properties = co_await file.Properties().GetImagePropertiesAsync();
        SimplePhotos::ImageFileInfo info(properties,
            file, file.DisplayName(), file.DisplayType());
        co_return info;
    }

    MainWindow::MainWindow() :
        m_images(winrt::single_threaded_observable_vector<IInspectable>())
    {
        InitializeComponent();
        GetItemsAsync();
    }


    void MainWindow::ImageGridView_ContainerContentChanging(ListViewBase const& sender, ContainerContentChangingEventArgs const& args) {
        if (args.InRecycleQueue())
        {
            auto templateRoot = args.ItemContainer().ContentTemplateRoot().try_as<Grid>();
            auto image = templateRoot.FindName(L"ItemImage").try_as<Image>();
            image.Source(nullptr);
        }
        if (args.Phase() == 0)
        {
            args.RegisterUpdateCallback({ this, &MainWindow::ShowImage });
            args.Handled(true);
        }
    }
    fire_and_forget MainWindow::ShowImage(ListViewBase const& sender, ContainerContentChangingEventArgs const& args)
    {
        if (args.Phase() == 1)
        {
            auto templateRoot = args.ItemContainer().ContentTemplateRoot().try_as<Grid>();
            auto image = templateRoot.FindName(L"ItemImage").try_as<Image>();
            auto item = args.Item().try_as<SimplePhotos::ImageFileInfo>();
            image.Source(co_await get_self<SimplePhotos::implementation::ImageFileInfo>(item)->GetImageThumbnailAsync());
        }
    }
}
```

```
//MainWindow.xaml.h
namespace winrt::SimplePhotos::implementation
{
   
    struct MainWindow : MainWindowT<MainWindow>
    {
        
        MainWindow();

            void ImageGridView_ContainerContentChanging(
                Microsoft::UI::Xaml::Controls::ListViewBase const& sender,
                Microsoft::UI::Xaml::Controls::ContainerContentChangingEventArgs const& args);

    public:
        
            fire_and_forget ShowImage(
                Microsoft::UI::Xaml::Controls::ListViewBase const& sender,
                Microsoft::UI::Xaml::Controls::ContainerContentChangingEventArgs const& args);
    
    
       
        Windows::Foundation::Collections::IVector<Windows::Foundation::IInspectable> Images() const
        {
            return m_images;
        }

    public:
        Windows::Foundation::Collections::IVector<IInspectable> m_images{ nullptr };

        Windows::Foundation::IAsyncAction GetItemsAsync();
        Windows::Foundation::IAsyncOperation<SimplePhotos::ImageFileInfo> LoadImageInfoAsync(Windows::Storage::StorageFile);
    
      
    };
}
```

실행 화면
![Image description](./simplephotos.PNG)</br>

