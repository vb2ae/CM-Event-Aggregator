# Caliburn.Micro 4 EventAggregator

I was looking at some of the Caliburn Micro issues. https://github.com/Caliburn-Micro/Caliburn.Micro/issues/755 was looking for example on how to do best way to fire an Event Message during BaseScreen's OnActivateAsync method

In this example I will have the shell view show a message on the MainView loading status

Here is the ShellView.xaml

    <Window x:Class="Setup.WPF.Views.ShellView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             xmlns:cm="http://caliburnmicro.com"
             mc:Ignorable="d"
            Title="ShellView" Height="300" Width="300">
        <Grid>
            <Grid.RowDefinitions>
                <RowDefinition Height="25"></RowDefinition>
                <RowDefinition></RowDefinition>
            </Grid.RowDefinitions>
            <TextBlock Text="{Binding Message}"/>
            <ContentControl cm:View.Model="{Binding ActiveItem}" Grid.Row="1" Margin="40,20"/>
        </Grid>
    </Window>


In the ShellViewModel I create the EventAggregator and register to receive the Publish events on the UI thread.  When a message is recieved it will be shown in textblock


ShellViewModel.cs  

    using System;
    using System.Collections.Generic;
    using System.ComponentModel;
    using System.Net.Http;
    using System.Runtime.CompilerServices;
    using System.Threading;
    using System.Threading.Tasks;
    using System.Windows;
    using System.Windows.Controls;
    using System.Windows.Documents;
    using System.Windows.Threading;
    using Caliburn.Micro;

    namespace Setup.WPF.ViewModels
    {
        public class ShellViewModel : Conductor<IScreen>.Collection.OneActive, IHandle<string>
        {
           private string _message;

           public string Message
           {
               get => _message;
               set
               {
                   _message = value;
                   NotifyOfPropertyChange("Message");
               }
           }


           private IEventAggregator _eventAggregator;
           public ShellViewModel()
           {
                _eventAggregator = new EventAggregator();
                _eventAggregator.SubscribeOnUIThread(this);
                Task.Run(async () =>
                 {
                    await ActivateItemAsync(new MainViewModel(_eventAggregator));
                 });
           }


            public Task HandleAsync(string message, CancellationToken cancellationToken)
            {
                this.Message = message.ToString();
                return Task.CompletedTask;
            }


         }
     }


Here is the xaml for the MainView xaml

    <UserControl x:Class="Setup.WPF.Views.MainView"
                  xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                  xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
                  xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006" 
                  xmlns:d="http://schemas.microsoft.com/expression/blend/2008" 
                  xmlns:local="clr-namespace:Setup.WPF.Views"
                   mc:Ignorable="d" 
                   d:DesignHeight="450" d:DesignWidth="800">
        <Grid>
            <TextBlock Text="{Binding Title}"/>
        </Grid>
    </UserControl>
    
The code for MainViewModel.  It registers to publish events when the ViewModel loads.  When the ViewIsActivating it sends a loading message.  To simulate a long running task it uses Task.Delay to pause a few seconds. 
hen it clears the loading message

    
    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Text;
    using System.Threading;
    using System.Threading.Tasks;
    using Caliburn.Micro;

    namespace Setup.WPF.ViewModels
    {
        public class MainViewModel : Screen
        {
            private string _title;

            public string Title
            {
                 get => _title;
                 set
                 {
                    _title = value;
                    NotifyOfPropertyChange("Title");
                  }
             }


            private IEventAggregator _eventAggregator;

             public MainViewModel(IEventAggregator eventAgg)
            {
                Title = "Welcome to Caliburn Micro in WPF";
                _eventAggregator = eventAgg;
              }

 
            protected override async Task OnDeactivateAsync(bool close, CancellationToken cancellationToken)
            {
                 await _eventAggregator.PublishOnUIThreadAsync("Closing");
            }

             protected override async Task OnActivateAsync(CancellationToken cancellationToken)
            {
                 await _eventAggregator.PublishOnUIThreadAsync("Loading");
                 await Task.Delay(2000);
                 await _eventAggregator.PublishOnUIThreadAsync(string.Empty);

            }


         }
     }

