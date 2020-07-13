# 启用spice\(Q版\)

#### 背景

OpenStack默认无法通过spice客户端直连，但实际上运行的qemu实例启用了spice，用以支持spice-html5。  
通过查看运行的qemu实例进程信息，就能看到如下实例的虚拟机使用的spice端口为6001

```text
/usr/libexec/qemu\-kvm -name guest=instance-000007c3 -vnc 0.0.0.0:100 -k en-us -spice port=6001,addr=0.0.0.0,disable-ticketing,seamless-migration=on
```

所以，通过客户端连接虚拟机理论上是可行的。

#### 操作步骤：

**修改计算节点防火墙**

通过防火墙规则允许spice的端口范围，可从5900开始开放一个范围，允许外界访问。  
此时通过spice客户端输入地址 spice://ip:port 就可以连接spice客户端了。

**连接安全**

spice直连没有密码，权限过大。qemu是支持spice设置密码的, 通过虚拟机实例的配置文件可以看到

```text
<graphics type='spice' autoport='yes' listen='0.0.0.0' keymap='en-us'>
      <listen type='address' address='0.0.0.0'/>
 </graphics>
```

但遗憾的是OpenStack创建虚拟机时并没有接口可以设置password选项。要增加password选项，需要修改OpenStack源码。需要修改两个文件。

1. `nova/virt/libvirt/driver.py` 通过在创建虚拟机时设置元数据`spice_passwd`项，并在创建虚拟机时使用该项完成。需要修改`_guest_add_video_device`方法，关键代码

```text
def _guest_add_video_device(guest, instance):
        # NB some versions of libvirt support both SPICE and VNC
        # at the same time. We're not trying to second guess which
        # those versions are. We'll just let libvirt report the
        # errors appropriately if the user enables both.
        add_video_driver = False
        if CONF.vnc.enabled and guest.virt_type not in ('lxc', 'uml'):
            graphics = vconfig.LibvirtConfigGuestGraphics()
            graphics.type = "vnc"
            if CONF.vnc.keymap:
                # TODO(stephenfin): There are some issues here that may
                # necessitate deprecating this option entirely in the future.
                # Refer to bug #1682020 for more information.
                graphics.keymap = CONF.vnc.keymap
            graphics.listen = CONF.vnc.server_listen
            guest.add_device(graphics)
            add_video_driver = True
        if CONF.spice.enabled and guest.virt_type not in ('lxc', 'uml', 'xen'):
            graphics = vconfig.LibvirtConfigGuestGraphics()
            graphics.type = "spice"
            if CONF.spice.keymap:
                # TODO(stephenfin): There are some issues here that may
                # necessitate deprecating this option entirely in the future.
                # Refer to bug #1682020 for more information.
                graphics.keymap = CONF.spice.keymap
            graphics.listen = CONF.spice.server_listen
            spice_passwd = instance.metadata.get('spice_passwd')
            if spice_passwd:
                graphics.passwd = spice_passwd
            guest.add_device(graphics)
            add_video_driver = True
        return add_video_driver
```

1. `nova/virt/libvirt/config.py` 需要修改`LibvirtConfigGuestGraphics`，关键代码

```text
class LibvirtConfigGuestGraphics(LibvirtConfigGuestDevice):

    def __init__(self, **kwargs):
        super(LibvirtConfigGuestGraphics, self).__init__(root_name="graphics",
                                                         **kwargs)

        self.type = "vnc"
        self.autoport = True
        self.keymap = None
        self.listen = None
        self.passwd = None

    def format_dom(self):
        dev = super(LibvirtConfigGuestGraphics, self).format_dom()

        dev.set("type", self.type)
        if self.autoport:
            dev.set("autoport", "yes")
        else:
            dev.set("autoport", "no")
        if self.keymap:
            dev.set("keymap", self.keymap)
        if self.listen:
            dev.set("listen", self.listen)
        if self.passwd:
            dev.set("passwd", self.passwd)
        return dev
```

**扩展API，通过API获取连接信息**

1. 新增API，增加`nova/api/openstack/compute/server_connectinfo.py`

```text
from webob import exc

from nova.api.openstack import common
from nova.api.openstack.compute.schemas import server_metadata
from nova.api.openstack import wsgi
from nova.api import validation
from nova import compute
from nova import exception
from nova.i18n import _
from nova.compute import rpcapi as compute_rpcapi

#ALIAS = 'os-server-connectinfo'
#authorize = extensions.os_compute_authorizer(ALIAS)


class ServerConnectinfoController(wsgi.Controller):
    """The server connect info API controller for the OpenStack API."""

    def __init__(self, *args, **kwargs):
        self.compute_api = compute.API()
        self.compute_rpcapi = compute_rpcapi.ComputeAPI()
        self.handlers = {'vnc': self.compute_rpcapi.get_vnc_console,
                         'spice': self.compute_rpcapi.get_spice_console,
                         'rdp': self.compute_rpcapi.get_rdp_console,
                         'serial': self.compute_rpcapi.get_serial_console,
                         'mks': self.compute_rpcapi.get_mks_console}
        #super(ServerConnectinfoController, self).__init__()
        super(ServerConnectinfoController, self).__init__(*args, **kwargs)

    def _get_connect_info(self, context, server_id, body):
        instance = common.get_instance(self.compute_api, context, server_id)
        try:
            protocol = body['os-getServerConnectinfo']['protocol']
            console_type = body['os-getServerConnectinfo']['type']
            handler = self.handlers.get(protocol)
            connect_info = handler(context,
                instance=instance, console_type=console_type)
        except exception.InstanceNotFound:
            msg = _('Server does not exist')
            raise exc.HTTPNotFound(explanation=msg)
        
        return connect_info

    @wsgi.Controller.api_version("2.1", "2.5")
    @wsgi.expected_errors((404))
    @wsgi.action('os-getServerConnectinfo')
    def index(self, req, id):
        """Returns the list of connectinfo for a given instance."""
        context = req.environ['nova.context']
        #authorize(context, action='index')
        return {'connectinfo': self._get_connect_info(context, id)}

    @wsgi.Controller.api_version("2.1", "2.5")
    @wsgi.expected_errors((404))
    @wsgi.action('os-getServerConnectinfo')
    def create(self, req, id, body):
        """Return a single connectinfo item."""
        context = req.environ['nova.context']
        #authorize(context, action='create')
        data = self._get_connect_info(context, id, body)

        try:
            return {'connectinfo': data}
        except KeyError:
            msg = _("Connectinfo item was not found")
            raise exc.HTTPNotFound(explanation=msg)
```

1. 在`nova/api/openstack/compute/routers.py`注册API

```text
from nova.api.openstack.compute import server_connectinfo


server_controller = functools.partial(_create_controller,
    servers.ServersController,
    [
        config_drive.ConfigDriveController,
        extended_availability_zone.ExtendedAZController,
        extended_server_attributes.ExtendedServerAttributesController,
        extended_status.ExtendedStatusController,
        extended_volumes.ExtendedVolumesController,
        hide_server_addresses.Controller,
        keypairs.Controller,
        security_groups.SecurityGroupsOutputController,
        server_usage.ServerUsageController,
    ],
    [
        admin_actions.AdminActionsController,
        admin_password.AdminPasswordController,
        console_output.ConsoleOutputController,
        create_backup.CreateBackupController,
        deferred_delete.DeferredDeleteController,
        evacuate.EvacuateController,
        floating_ips.FloatingIPActionController,
        lock_server.LockServerController,
        migrate_server.MigrateServerController,
        multinic.MultinicController,
        pause_server.PauseServerController,
        remote_consoles.RemoteConsolesController,
        rescue.RescueController,
        security_groups.SecurityGroupActionController,
        shelve.ShelveController,
        suspend_server.SuspendServerController,
        server_connectinfo.ServerConnectinfoController,
    
    ]
)
```

### 验证控制节点API

#### 先获取token

{% page-ref page="../keystone/ming-ling-hang-huo-qu-token.md" %}



#### 请求连接方式

```text

  curl -XPOST \
  --url http://{nova-compute-api-endpoint}/v2.1/servers/e691fe62-62c9-4d0b-8420-a8d0227b518e/action \
  --header 'cache-control: no-cache' \
  --header 'content-type: application/json' \
  --header 'postman-token: 4d0f56ee-83a4-742c-9b0d-3403d44bb49e' \
  --header 'x-auth-token: gAAAAABfDGVZ_Wiwb2Mr5EVluJyDrFEhH4D0rAxNBDKUGx6sF37-88P8j2MWgUVK6hzjz-0jNfFoNfqyZEqooj2aEXAq3Zj9x2Q4jEYQXdgMF_PuhqWEgSVNOodePe6SGcgI4EqV4d2dR5EWvr92kUsVLLKsmuEmVNvskFdzwoho5f0hAx3pgT4' \
  --data '{ "os-getServerConnectinfo": {     "protocol": "spice",     "type": "spice-html5" }}'
```

