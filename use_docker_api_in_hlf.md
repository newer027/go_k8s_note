# 调用docker api的开源软件相关代码分析

-internal/peer/node/start.go
```
func serve(args []string) error {
	go chaincodeCustodian.Work(buildRegistry, containerRouter, chaincodeLauncher)
}
```

- core/chaincode/lifecycle/custodian.go
```
func (cc *ChaincodeCustodian) Work(buildRegistry *container.BuildRegistry, builder ChaincodeBuilder, launcher ChaincodeLauncher) {
	for {
		cc.mutex.Lock()
		if len(cc.choreQueue) == 0 && !cc.halt {
			cc.cond.Wait()
		}
		if cc.halt {
			cc.mutex.Unlock()
			return
		}
		chore := cc.choreQueue[0]
		cc.choreQueue = cc.choreQueue[1:]
		cc.mutex.Unlock()

		if chore.runnable {
			if err := launcher.Launch(chore.chaincodeID); err != nil {
				logger.Warningf("could not launch chaincode '%s': %s", chore.chaincodeID, err)
			}
		} else {
			buildStatus, ok := buildRegistry.BuildStatus(chore.chaincodeID)
			if ok {
				logger.Debugf("skipping build of chaincode '%s' as it is already in progress", chore.chaincodeID)
				continue
			}
			err := builder.Build(chore.chaincodeID)
			if err != nil {
				logger.Warningf("could not build chaincode '%s': %s", chore.chaincodeID, err)
			}
			buildStatus.Notify(err)
		}
	}
}
```

- core/chaincode/runtime_launcher.go
```
func (r *RuntimeLauncher) Launch(ccid string) error {
	var startFailCh chan error
	var timeoutCh <-chan time.Time

	startTime := time.Now()
	launchState, alreadyStarted := r.Registry.Launching(ccid)
	if !alreadyStarted {
		startFailCh = make(chan error, 1)
		timeoutCh = time.NewTimer(r.StartupTimeout).C

		go func() {
			var ccservinfo *ccintf.ChaincodeServerInfo
			ccservinfo, err := r.Runtime.Build(ccid)
			if err != nil {
				startFailCh <- errors.WithMessage(err, "error building chaincode")
				return
			}
			if ccservinfo != nil {
				startFailCh <- errors.New("peer as client to be implemented")
				return
			}
			ccinfo, err := r.ChaincodeClientInfo(ccid)
			if err != nil {
				startFailCh <- errors.WithMessage(err, "could not get connection info")
				return
			}
			if ccinfo == nil {
				startFailCh <- errors.New("could not get connection info")
				return
			}
			if err = r.Runtime.Start(ccid, ccinfo); err != nil {
				startFailCh <- errors.WithMessage(err, "error starting container")
				return
			}
			exitCode, err := r.Runtime.Wait(ccid)
			if err != nil {
				launchState.Notify(errors.Wrap(err, "failed to wait on container exit"))
			}
			launchState.Notify(errors.Errorf("container exited with %d", exitCode))
		}()
	}

	var err error
	select {
	case <-launchState.Done():
		err = errors.WithMessage(launchState.Err(), "chaincode registration failed")
	case err = <-startFailCh:
		launchState.Notify(err)
		r.Metrics.LaunchFailures.With("chaincode", ccid).Add(1)
	case <-timeoutCh:
		err = errors.Errorf("timeout expired while starting chaincode %s for transaction", ccid)
		launchState.Notify(err)
		r.Metrics.LaunchTimeouts.With("chaincode", ccid).Add(1)
	}

	success := true
	if err != nil && !alreadyStarted {
		success = false
		chaincodeLogger.Debugf("stopping due to error while launching: %+v", err)
		defer r.Registry.Deregister(ccid)
		if err := r.Runtime.Stop(ccid); err != nil {
			chaincodeLogger.Debugf("stop failed: %+v", err)
		}
	}

	r.Metrics.LaunchDuration.With(
		"chaincode", ccid,
		"success", strconv.FormatBool(success),
	).Observe(time.Since(startTime).Seconds())

	chaincodeLogger.Debug("launch complete")
	return err
}
```

- core/chaincode/container_runtime.go
```
// Start launches chaincode in a runtime environment.
func (c *ContainerRuntime) Start(ccid string, ccinfo *ccintf.PeerConnection) error {
	chaincodeLogger.Debugf("start container: %s", ccid)

	if err := c.ContainerRouter.Start(ccid, ccinfo); err != nil {
		return errors.WithMessage(err, "error starting container")
	}

	return nil
}
```

- core/container/container.go
```
func (r *Router) Start(ccid string, peerConnection *ccintf.PeerConnection) error {
	return r.getInstance(ccid).Start(peerConnection)
}
```

- core/container/dockercontroller/dockercontroller.go
```
func (ci *ContainerInstance) Start(peerConnection *ccintf.PeerConnection) error {
	return ci.DockerVM.Start(ci.CCID, ci.Type, peerConnection)
}
```


```
// Start starts a container using a previously created docker image
func (vm *DockerVM) Start(ccid string, ccType string, peerConnection *ccintf.PeerConnection) error {
	imageName, err := vm.GetVMNameForDocker(ccid)
	if err != nil {
		return err
	}

	containerName := vm.GetVMName(ccid)
	logger := dockerLogger.With("imageName", imageName, "containerName", containerName)

	vm.stopInternal(containerName)

	args, err := vm.GetArgs(ccType, peerConnection.Address)
	if err != nil {
		return errors.WithMessage(err, "could not get args")
	}
	dockerLogger.Debugf("start container with args: %s", strings.Join(args, " "))

	env := vm.GetEnv(ccid, peerConnection.TLSConfig)
	dockerLogger.Debugf("start container with env:\n\t%s", strings.Join(env, "\n\t"))

	err = vm.createContainer(imageName, containerName, args, env)
	if err != nil {
		logger.Errorf("create container failed: %s", err)
		return err
	}

	// stream stdout and stderr to chaincode logger
	if vm.AttachStdOut {
		containerLogger := flogging.MustGetLogger("peer.chaincode." + containerName)
		streamOutput(dockerLogger, vm.Client, containerName, containerLogger)
	}

	// upload TLS files to the container before starting it if needed
	if peerConnection.TLSConfig != nil {
		// the docker upload API takes a tar file, so we need to first
		// consolidate the file entries to a tar
		payload := bytes.NewBuffer(nil)
		gw := gzip.NewWriter(payload)
		tw := tar.NewWriter(gw)

		// Note, we goofily base64 encode 2 of the TLS artifacts but not the other for strange historical reasons
		err = addFiles(tw, map[string][]byte{
			TLSClientKeyPath:      []byte(base64.StdEncoding.EncodeToString(peerConnection.TLSConfig.ClientKey)),
			TLSClientCertPath:     []byte(base64.StdEncoding.EncodeToString(peerConnection.TLSConfig.ClientCert)),
			TLSClientKeyFile:      peerConnection.TLSConfig.ClientKey,
			TLSClientCertFile:     peerConnection.TLSConfig.ClientCert,
			TLSClientRootCertFile: peerConnection.TLSConfig.RootCert,
		})
		if err != nil {
			return fmt.Errorf("error writing files to upload to Docker instance into a temporary tar blob: %s", err)
		}

		// Write the tar file out
		if err := tw.Close(); err != nil {
			return fmt.Errorf("error writing files to upload to Docker instance into a temporary tar blob: %s", err)
		}

		gw.Close()

		err := vm.Client.UploadToContainer(containerName, docker.UploadToContainerOptions{
			InputStream:          bytes.NewReader(payload.Bytes()),
			Path:                 "/",
			NoOverwriteDirNonDir: false,
		})
		if err != nil {
			return fmt.Errorf("Error uploading files to the container instance %s: %s", containerName, err)
		}
	}
    // start container with HostConfig was deprecated since v1.10 and removed in v1.2
	err = vm.Client.StartContainer(containerName, nil)
	if err != nil {
		dockerLogger.Errorf("start-could not start container: %s", err)
		return err
	}

	dockerLogger.Debugf("Started container %s", containerName)
	return nil
}
```

- core/container/dockercontroller/dockercontroller.go
```
func (vm *DockerVM) createContainer(imageID, containerID string, args, env []string) error {
	logger := dockerLogger.With("imageID", imageID, "containerID", containerID)
	logger.Debugw("create container")
	_, err := vm.Client.CreateContainer(docker.CreateContainerOptions{
		Name: containerID,
		Config: &docker.Config{
			Cmd:          args,
			Image:        imageID,
			Env:          env,
			AttachStdout: vm.AttachStdOut,
			AttachStderr: vm.AttachStdOut,
		},
		HostConfig: vm.HostConfig,
	})
	if err != nil {
		return err
	}
	logger.Debugw("created container")
	return nil
}

```

- vendor/github.com/fsouza/go-dockerclient/container.go
```
// CreateContainer creates a new container, returning the container instance,
// or an error in case of failure.
//
// The returned container instance contains only the container ID. To get more
// details about the container after creating it, use InspectContainer.
//
// See https://goo.gl/tyzwVM for more details.
func (c *Client) CreateContainer(opts CreateContainerOptions) (*Container, error) {
	path := "/containers/create?" + queryString(opts)
	resp, err := c.do(
		"POST",
		path,
		doOptions{
			data: struct {
				*Config
				HostConfig       *HostConfig       `json:"HostConfig,omitempty" yaml:"HostConfig,omitempty" toml:"HostConfig,omitempty"`
				NetworkingConfig *NetworkingConfig `json:"NetworkingConfig,omitempty" yaml:"NetworkingConfig,omitempty" toml:"NetworkingConfig,omitempty"`
			}{
				opts.Config,
				opts.HostConfig,
				opts.NetworkingConfig,
			},
			context: opts.Context,
		},
	)

	if e, ok := err.(*Error); ok {
		if e.Status == http.StatusNotFound && strings.Contains(e.Message, "No such image") {
			return nil, ErrNoSuchImage
		}
		if e.Status == http.StatusConflict {
			return nil, ErrContainerAlreadyExists
		}
		// Workaround for 17.09 bug returning 400 instead of 409.
		// See https://github.com/moby/moby/issues/35021
		if e.Status == http.StatusBadRequest && strings.Contains(e.Message, "Conflict.") {
			return nil, ErrContainerAlreadyExists
		}
	}

	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()
	var container Container
	if err := json.NewDecoder(resp.Body).Decode(&container); err != nil {
		return nil, err
	}

	container.Name = opts.Name

	return &container, nil
}
```