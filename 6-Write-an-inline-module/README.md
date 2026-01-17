# CruzHacks 2026 Rover Inline Module Tutorial

This tutorial shows how to build a Viam inline module that makes a rover spin right, pause, then spin left. After developing it, you can run the sequence in the Viam UI, or implement a scheduled job to automate it.

The module behavior is controlled by three attributes:

- `right_degrees` (float): how far to spin to the right.
- `pause_duration` (float): how long to pause in seconds.
- `left_degrees` (float): how far to spin to the left.

You will also declare a single dependency:

- `base` (string): the name of the base component on your rover.

## How to use inline modules

1. Navigate to your machine's page.
1. In the left-hand menu of the **CONFIGURE** tab, click the **+** (Create) icon.
![`create menu`](./assets/createMenu.png)
1. Select **Control code**.
1. Choose the **Viam-hosted** option.
![`viam hosted option`](./assets/viamHostedOption.png)
1. Name the module after your project and choose the desired programming language. Click **Create** to generate the module.
![`create inline module`](./assets/createInlineModule.png)
1. In the code editor, paste in the fully working code example below or write your own logic here.
1. Click **Save & Deploy** to build the module. The module will take a few minutes to build, and then it is ready to test on a machine.
![`save and deploy`](./assets/saveAndDeploy.png)
1. When the build has completed successfully, click the **Add to machine** button, find your machine and add the module to it.
![`add to machine`](./assets/addToMachine.png)
![`select your machine`](./assets/selectYourMachine.png)
1. Click the Add button in the module section of your machine to add the generic service to your machine. Give it a name or use the default name.
![`add generic service`](./assets/addServiceToMachine.png)
1. Fill in the attributes JSON the attributes and add your dependencies (examples below).
1. **Save** your config.
![`save config`](./assets/saveChanges.png)
1. Use the **DoCommand** test panel and click **Execute** (empty payload) to run the spin sequence.
1. Finally, add a scheduled job to your machine to automate the task. In the left-hand menu of the **CONFIGURE** tab, click the **+** (Create) icon, and choose **Job**. Given the job a name or use the default name.
1. Set the desired schedule for your job, and select the generic service as the task to run.
![`schedule job`](./assets/scheduledJob.png)
1. **Save** your config.
![`save config`](./assets/saveChanges.png)


## Attributes and dependencies (example JSON)

Generic service attributes:

```json
{
  "right_degrees": 90,
  "pause_duration": 1.5,
  "left_degrees": 90,
  "base": "base"
}
```

Notes:

- The `base` attribute is both a dependency and a required attribute. It should match the name of your base component in the machine config.
- The module uses a default spin speed of 90 degrees per second. Adjust it in code if desired.

---

## Python inline module (pasted into the code editor)

```python
import asyncio
from typing import ClassVar, Mapping, Optional, Sequence, Tuple

from typing_extensions import Self
from viam.components.base import Base
from viam.proto.app.robot import ComponentConfig
from viam.proto.common import ResourceName
from viam.resource.base import ResourceBase
from viam.resource.easy_resource import EasyResource
from viam.resource.types import Model, ModelFamily
from viam.services.generic import *
from viam.utils import ValueTypes
from viam.utils import struct_to_dict

class GenericService(Generic, EasyResource):
    MODEL: ClassVar[Model] = Model(ModelFamily("viam", "cruzhacks"), "spin_pause")

    @classmethod
    def new(
        cls, config: ComponentConfig, dependencies: Mapping[ResourceName, ResourceBase]
    ) -> Self:
        self = super().new(config, dependencies)

        attrs = struct_to_dict(config.attributes)
        self.right_degrees = float(attrs.get("right_degrees"))
        self.pause_duration = float(attrs.get("pause_duration"))
        self.left_degrees = float(attrs.get("left_degrees"))
        self.spin_speed_deg_per_sec = float(attrs.get("spin_speed_deg_per_sec", 90))

        base_name = attrs.get("base")
        self.base = dependencies[Base.get_resource_name(base_name)]

        return self

    @classmethod
    def validate_config(
        cls, config: ComponentConfig
    ) -> Tuple[Sequence[str], Sequence[str]]:
        attrs = struct_to_dict(config.attributes)

        right_degrees = attrs.get("right_degrees")
        if right_degrees is None or not isinstance(right_degrees, (int, float)):
            raise ValueError(
                "attribute 'right_degrees' is required and must be a number"
            )

        pause_duration = attrs.get("pause_duration")
        if pause_duration is None or not isinstance(pause_duration, (int, float)):
            raise ValueError(
                "attribute 'pause_duration' is required and must be a number"
            )

        left_degrees = attrs.get("left_degrees")
        if left_degrees is None or not isinstance(left_degrees, (int, float)):
            raise ValueError(
                "attribute 'left_degrees' is required and must be a number"
            )

        base_name = attrs.get("base")
        if not isinstance(base_name, str) or not base_name:
            raise ValueError("attribute 'base' (non-empty string) is required")

        return [base_name], []

    async def do_command(
        self,
        command: Mapping[str, ValueTypes],
        *,
        timeout: Optional[float] = None,
        **kwargs
    ) -> Mapping[str, ValueTypes]:
        # Positive degrees spin right, negative degrees spin left.
        await self.base.spin(self.right_degrees, self.spin_speed_deg_per_sec)
        await asyncio.sleep(self.pause_duration)
        await self.base.spin(-self.left_degrees, self.spin_speed_deg_per_sec)
        return {
            "right_degrees": self.right_degrees,
            "pause_duration": self.pause_duration,
            "left_degrees": self.left_degrees,
            "status": "completed",
        }
```

---

## Go inline module (inline editor code)

```go
package inlinegenericservice

import (
	"context"
	"errors"
	"time"

	"go.viam.com/rdk/components/base"
	"go.viam.com/rdk/logging"
	"go.viam.com/rdk/resource"
	generic "go.viam.com/rdk/services/generic"
)

var (
	GenericService = resource.NewModel("viam", "cruzhacks", "spin_pause")
)

func init() {
	resource.RegisterService(generic.API, GenericService,
		resource.Registration[resource.Resource, *Config]{
			Constructor: newGenericService,
		},
	)
}

type Config struct {
	RightDegrees       float64  `json:"right_degrees"`
	PauseDuration      float64  `json:"pause_duration"`
	LeftDegrees        float64  `json:"left_degrees"`
	SpinSpeedDegPerSec *float64 `json:"spin_speed_deg_per_sec,omitempty"`
	Base               string   `json:"base"`
}

func (cfg *Config) Validate(path string) ([]string, []string, error) {
	if cfg.RightDegrees == 0 {
		return nil, nil, errors.New(path + ": attribute 'right_degrees' is required")
	}
	if cfg.PauseDuration == 0 {
		return nil, nil, errors.New(path + ": attribute 'pause_duration' is required")
	}
	if cfg.LeftDegrees == 0 {
		return nil, nil, errors.New(path + ": attribute 'left_degrees' is required")
	}
	if cfg.Base == "" {
		return nil, nil, errors.New(path + ": attribute 'base' (non-empty string) is required")
	}
	return []string{cfg.Base}, []string{}, nil
}

type genericService struct {
	resource.AlwaysRebuild

	name   resource.Name
	logger logging.Logger
	cfg    *Config

	cancelCtx  context.Context
	cancelFunc func()

	base               base.Base
	rightDegrees       float64
	pauseDuration      float64
	leftDegrees        float64
	spinSpeedDegPerSec float64
}

func newGenericService(ctx context.Context, deps resource.Dependencies, rawConf resource.Config, logger logging.Logger) (resource.Resource, error) {
	conf, err := resource.NativeConfig[*Config](rawConf)
	if err != nil {
		return nil, err
	}

	return NewGenericService(ctx, deps, rawConf.ResourceName(), conf, logger)
}

func NewGenericService(ctx context.Context, deps resource.Dependencies, name resource.Name, conf *Config, logger logging.Logger) (resource.Resource, error) {
	cancelCtx, cancelFunc := context.WithCancel(context.Background())

	baseDep, err := base.FromDependencies(deps, conf.Base)
	if err != nil {
		return nil, err
	}

	spinSpeed := float64(90)
	if conf.SpinSpeedDegPerSec != nil {
		spinSpeed = *conf.SpinSpeedDegPerSec
	}

	s := &genericService{
		name:               name,
		logger:             logger,
		cfg:                conf,
		cancelCtx:          cancelCtx,
		cancelFunc:         cancelFunc,
		base:               baseDep,
		rightDegrees:       conf.RightDegrees,
		pauseDuration:      conf.PauseDuration,
		leftDegrees:        conf.LeftDegrees,
		spinSpeedDegPerSec: spinSpeed,
	}
	return s, nil
}

func (s *genericService) Name() resource.Name {
	return s.name
}

func (s *genericService) DoCommand(ctx context.Context, cmd map[string]interface{}) (map[string]interface{}, error) {
	if err := s.base.Spin(ctx, s.rightDegrees, s.spinSpeedDegPerSec, nil); err != nil {
		return nil, err
	}
	time.Sleep(time.Duration(s.pauseDuration * float64(time.Second)))
	if err := s.base.Spin(ctx, -s.leftDegrees, s.spinSpeedDegPerSec, nil); err != nil {
		return nil, err
	}
	return map[string]interface{}{
		"right_degrees":  s.rightDegrees,
		"pause_duration": s.pauseDuration,
		"left_degrees":   s.leftDegrees,
		"status":         "completed",
	}, nil
}

func (s *genericService) Close(context.Context) error {
	s.cancelFunc()
	return nil
}
```

---

## Testing tips

- Use the generic service **DoCommand** panel and click **Execute** with an empty payload.
- If the rover spins the wrong direction, swap the sign in the two `Spin` calls or swap your `right_degrees` and `left_degrees` values.
- If a spin fails, verify the base dependency name matches the base component name in your robot config.

