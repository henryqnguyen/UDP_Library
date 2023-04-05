# UDP_Library

```c#

/// HQN: this method send data to atsd atalin inside the file: Atalin.cs

    private async Task UpdateATSDEmulatorAsync()
    {
      _sendingEndPoint ??= new IPEndPoint(IPAddress.Parse(atsdAtalinEmulatorAddr), Convert.ToInt32(atsdAtalinEmulatorSendPort));

      try
      {
        byte[] data = new byte[16];
        float vehicleAz = (Position.ToEarthAzimuth(_azimuthUpdated) - Core.Instance.Orientation.Yaw + 9600) % 6400f;
        Buffer.BlockCopy(BitConverter.GetBytes(IPAddress.HostToNetworkOrder((int)(vehicleAz * 1000))), 0, data, 0, 4);
        Buffer.BlockCopy(BitConverter.GetBytes(IPAddress.HostToNetworkOrder((int)(_elevationUpdated * 1000))), 0, data, 4, 4);
        Buffer.BlockCopy(BitConverter.GetBytes(IPAddress.HostToNetworkOrder((int)(1 * 1000))), 0, data, 8, 4);
        Buffer.BlockCopy(BitConverter.GetBytes(IPAddress.HostToNetworkOrder((int)(1 * 1000))), 0, data, 12, 4);

        //HQN = This works
        //_talinUDPConnection.Send(data, Properties.Settings.Default.EmulatorAddress, Properties.Settings.Default.EmulatorSendPort);

        //HQN = This works
        //_udpClient.Send(data, data.Length, _sendingEndPoint);

        //_udpClient.Client.SendTo(data, 0, data.Length, SocketFlags.None, _sendingEndPoint);

        await _udpClient.SendAsync(data, data.Length, _sendingEndPoint);
      }
      catch (Exception ex)
      {
        _sendingEndPoint = null;
        Console.WriteLine("[Atalin](SendUpdate) ERROR Exception");
        Console.WriteLine(ex.Message);
      }
    }
        
    ```
    
    
    **This is receive UDP messages**
    
    ```csharp
        private async void ReceiveGunDriveDataAsync()
    {
      _receivingEndPoint ??= new IPEndPoint(IPAddress.Any, Int32.Parse(Properties.Settings.Default.EmulatorListenPort));

      if (_receivingEndPoint == null)
      {
        _receivingEndPoint = new IPEndPoint(IPAddress.Any, Int32.Parse(Properties.Settings.Default.EmulatorListenPort));

        //HQN: if EPCS Emulator IP is set at 192.168.1.6 and NOT receiving data then uncomment the line below 

        //_udpClient.Client.Bind(_receivingEndPoint);
      }

      _udpClient ??= new UdpClient(_receivingEndPoint);

      // Loop indefinitely to receive and process gun position data messages
      while (!stopButtonSelected)
      {
        try
        {
          // Receive the gun position data message
          UdpReceiveResult result = await _udpClient.ReceiveAsync();

          byte[] data = result.Buffer;

          // Reverse the byte order of the timestamp has no effect
          Array.Reverse(data, 4, 4);
          Array.Reverse(data, 8, 4);  // Reverse the byte order of the azimuth field
          Array.Reverse(data, 12, 4); // Reverse the byte order of the pitch field
          Array.Reverse(data, 16, 4); // Reverse the byte order of the roll field

          if (data.Length == 20)
          {
            // Extract the fields from the message using BitConverter
            UInt16 sequenceNumber = BitConverter.ToUInt16(data, 0);
            UInt16 validity = BitConverter.ToUInt16(data, 2);
            UInt32 timestamp = BitConverter.ToUInt32(data, 4);
            float azimuthMils = BitConverter.ToSingle(data, 8);
            float pitchMils = BitConverter.ToSingle(data, 12);
            float roll = BitConverter.ToSingle(data, 16);
            Console.WriteLine("received azimuthMils: " + azimuthMils);
            Console.WriteLine("received pitchMils: " + pitchMils);

            pitchMils -= Core.Instance.Orientation.Pitch;

            if (Core.Instance.Calibrations.Down.Value < pitchMils && pitchMils < Core.Instance.Calibrations.Up.Value)
            {
              if (!Core.Instance.Motion.Elevation.Valid)
              {
                if (pitchMils == 0 && azimuthMils == 3200)
                {

                }
                else
                {
                  Core.Instance.Position.Platform.Elevation = pitchMils;
                  Console.WriteLine("epcs el updated no move: " + Core.Instance.Position.Platform.Elevation);
                }
              }
              else if (Math.Round(pitchMils) == Math.Round(_elevationUpdated))
              {
                Core.Instance.Position.Platform.Elevation = pitchMils;
                Console.WriteLine("epcs el updated in move: " + Core.Instance.Position.Platform.Elevation);
              }
            }

            azimuthMils = Position.ToPlatformAzimuth(azimuthMils);

            if (Core.Instance.Calibrations.Left.Value < azimuthMils && azimuthMils < Core.Instance.Calibrations.Right.Value)
            {
              if (!Core.Instance.Motion.Azimuth.Valid)
              {
                if (pitchMils == 0 && azimuthMils == 3200)
                {

                }
                else
                {
                  Core.Instance.Position.Platform.Azimuth = azimuthMils;
                  Console.WriteLine("epcs az updated no move: " + Core.Instance.Position.Platform.Azimuth);
                }
              }
              else if (Math.Round(azimuthMils) == Math.Round(_azimuthUpdated))
              {
                Core.Instance.Position.Platform.Azimuth = azimuthMils;
                Console.WriteLine("epcs az updated no move: " + Core.Instance.Position.Platform.Azimuth);
              }
            }
          }

          else
          {
            Console.WriteLine("Received invalid gun position data message");
          }
        }
        catch (Exception e)
        {
          Console.WriteLine($"Error receiving gun position data message: {e.Message}");
        }
      }
    }
    ```
    
    
