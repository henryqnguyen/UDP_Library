# UDP_Library

```c#

/// HQN: this method send data to atsd atalin

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
