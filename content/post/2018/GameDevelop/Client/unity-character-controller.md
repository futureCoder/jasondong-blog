---
title: "使用Unity3d中的CharacterController进行碰撞检测"
date: 2018-04-24T09:33:53+08:00
draft: false
---

CharacterController角色控制器用于第一或第三人称主角控制，不实用刚体物理效果。继承自Collider，不受力的影响，通过调用Move函数进行移动，但是受制于碰撞。
CharacterController多用于“角色”上，因为“角色”是“活”的，他们的运动要么是玩家控制，要么是脚本控制，所以一般不需要物理系统来控制。如果由物理系统控制，反而会使游戏的操作性下降。由玩家控制的角色肯定不用```Rigidbody```。

当执行移动时，碰到一个```Collider```时，```OnControllerColliderHit(ControllerColliderHit hit)```被调用.下面是控制角色推动一个物体的代码。  

{{< codeblock "OnControllerColliderHit.cs">}}
private string castName = null;

private string receiveName = null;
void OnControllerColliderHit(ControllerColliderHit hit)
{
    //得到接收碰撞名称
    GameObject hitObject = hit.collider.gameObject;
    //当它不是地面时
    if (!hitObject.name.Equals("Plane"))
    {
        //得到主动碰撞的对象 与接收碰撞的对象名称
        castName = gameObject.name;
        receiveName = hitObject.name;
    }
    Rigidbody body = hit.collider.attachedRigidbody;
    //dont move the rigidbody if the character is on top of it

    if (body == null || body.isKinematic || !hitObject.name.Equals("Player (1)"))
    {
        return;
    }
    body.AddForce(new Vector3(hit.moveDirection.x, 0, hit.moveDirection.z) * 10);
}
{{< /codeblock >}}
