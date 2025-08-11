---
layout: post
title: Custom design-time attributes in Silverlight and WPF designer [C#]
date: 2011-08-15
tags:
  - csharp
series: asyncctp
---
We all know the standard [design-time attributes in the Silverlight](http://msdn.microsoft.com/en-us/library/ff602277%28v=vs.95%29.aspx) designer like d:DataContext, d:DesignWidth or d:DesignHeight. Let’s see how easy we can add some new attributes.

![designtime](/assets/images/designtime.png)


```csharp
using System.ComponentModel;  
using System.Windows;  
  
namespace DesignTimeProperties  
{  
    public static class d  
    {  
        static bool? inDesignMode;  
  
        /// <summary>  
        /// Indicates whether or not the framework is in design-time mode. (Caliburn.Micro implementation)  
        /// </summary>  
        private static bool InDesignMode  
        {  
            get  
            {  
                if (inDesignMode == null)  
                {  
                    var prop = DesignerProperties.IsInDesignModeProperty;  
                    inDesignMode = (bool)DependencyPropertyDescriptor.FromProperty(prop, typeof(FrameworkElement)).Metadata.DefaultValue;  
  
                    if (!inDesignMode.GetValueOrDefault(false) && System.Diagnostics.Process.GetCurrentProcess()  
                            .ProcessName.StartsWith("devenv", System.StringComparison.Ordinal))  
                        inDesignMode = true;  
                }  
  
                return inDesignMode.GetValueOrDefault(false);  
            }  
        }  
  
        public static DependencyProperty BackgroundProperty = DependencyProperty.RegisterAttached(  
            "Background", typeof(System.Windows.Media.Brush), typeof(d),  
            new PropertyMetadata(new PropertyChangedCallback(BackgroundChanged)));  
  
        public static System.Windows.Media.Brush GetBackground(DependencyObject dependencyObject)  
        {  
            return (System.Windows.Media.Brush)dependencyObject.GetValue(BackgroundProperty);  
        }  
        public static void SetBackground(DependencyObject dependencyObject, System.Windows.Media.Brush value)  
        {  
            dependencyObject.SetValue(BackgroundProperty, value);  
        }  
        private static void BackgroundChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)  
        {  
            if (!InDesignMode)  
                return;  
  
            d.GetType().GetProperty("Background").SetValue(d, e.NewValue, null);  
        }  
    }  
}
```

Great, but what about many others very useful properties ? Do we need to write them all manually? No, we don’t. I have prepared T4 template that generates code for all properties of all controls available in WPF and Silverlight assemblies. When you look carefully at the code above you will find that I am using the reflection to set the value of the property. It’s because some properties like Background are defined many times in many different controls so instead of checking the actual control type I assume that the control contains appropriate property. This assumption simplifies the implementation very much. And of course the property can be set only in design time when we are editing the form inside Expression Blend or Visual Studio. T4 template code analyzes all properties from all controls and properties with the same names but different types are skipped during generation process.

[Download](http://archive.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=mnajder&DownloadId=15650)