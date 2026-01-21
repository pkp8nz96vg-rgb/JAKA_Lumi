## SDK外部轴控制说明文档



### 上下使能

```C++
errno_t enable_ext(int ext_id);
errno_t disable_ext(int ext_id);
```



### 获取外部轴状态

```c++
errno_t get_ext_status(ExtAxisStatusList *status, int ext_id = -1);

typedef struct
{
    BOOL is_powered;    // 已上电
    BOOL is_powering;   // 正在上电
    BOOL is_enabled;    // 已使能
    BOOL is_enabling;   // 正在使能
    BOOL is_inpos;      // 已到位
    BOOL is_on_limit;   // 是否处于限位
    double pos_cmd;     //  目标位置
    double pos_fdb;     //  反馈位置
} ExtAxisStatus;

typedef struct
{
    int count;
    ExtAxisStatus status[MAX_EXT_NUMS];
}ExtAxisStatusList;
```



### 运动相关

#### 外部轴点动

```c++
errno_t jog_ext(int ext_id, int is_abs, double vel, double step);
// ext_id:外部轴ID
// is_abs: 0:ABS, 1:INCR, 2:CONTINUE
// vel: 速度百分比,INCR和CONTINUE运动方向由速度控制
// step：ABS:step为目标位置; INCR:step为目标步长; CONTINUE:step无意义
```

#### 组合运动

```c++
errno_t multi_mov_with_ext(MultiMovInfoList multi_mov_info_list, DI_Info *di_info, int plannertype);

typedef struct{
    int motion_unit_type;    //0:机器人, 1:外部轴
    int move_type;           //0:关节, 1:直线2圆弧 
    int motion_unit_id;      //运动单位id
    CartesianPose mid_pos;
    double end_pos[MAX_AXIS];//目标关节角度/末端位置
    int move_mode;           //0:绝对运动, 1:相对运动
    double j_vel;
    double j_acc;
    double j_jerk;
    double vel;
    double acc;
    double jerk;
    double ori_vel;
    double ori_acc;
    double ori_jerk;
    double radius;             
    double circle;
    int circle_mode;
}MultiMovInfo;
typedef struct {
    int count;     //运动组数量
    BOOL is_block; //0:非阻塞, 1:阻塞
    MultiMovInfo multi_mov_info[MAX_EXT];
} MultiMovInfoList;
```



#### 示例

#### 

```c++
#include <iostream>
#include <thread>
#include <math.h>
#include "JAKAZuRobot.h"  

#define rad2deg(x) ((x)*180.0/M_PI)
#define deg2rad(x) ((x)*M_PI/180.0)
void jog_ext();
void move_ext();
void movej_with_ext();
void movel_with_ext();
void movec_with_ext();
JAKAZuRobot robot;
errno_t ret;
int main()
{
    //初始化，上电上使能
    ret = robot.login_in("192.168.220.144",1,"jaka_sdk","Jaka123@");
    if(ret==0){std::cout << "登陆成功" << std::endl;}
    else {std::cout << "登陆失败" << std::endl;}
    robot.power_on();
    std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    robot.enable_robot();
    std::this_thread::sleep_for(std::chrono::milliseconds(1000));

    JointValue jpos = {0, deg2rad(90), deg2rad(90), deg2rad(90), deg2rad(-90), deg2rad(10)};
    robot.joint_move(&jpos, ABS, true, 10);
    std::cout << "初始位置完成" << std::endl;
    robot.enable_ext(0);
    std::cout << "enable_ext完成" << std::endl;
    jog_ext();
    move_ext();
    movej_with_ext();
    movel_with_ext();
    movec_with_ext();

    robot.login_out();
    return 0;
}

void jog_ext()
{
    std::cout << " jog_ext" << std::endl; 
    robot.set_auto_manual_mode(false);
    std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    robot.jog_ext(0,0,80,5);
    std::this_thread::sleep_for(std::chrono::milliseconds(2000));
    robot.jog_ext(0,0,80,4);
    std::this_thread::sleep_for(std::chrono::milliseconds(1000));

    robot.jog_stop(-1);
    robot.jog_ext(0,1,80,1);
    std::this_thread ::sleep_for(std::chrono::milliseconds(1000));
    robot.jog_stop(-1);
    robot.jog_ext(0,1,-80,1);
    std::this_thread::sleep_for(std::chrono::milliseconds(1000));

    robot.jog_ext(0,2,1,0);
    std::this_thread::sleep_for(std::chrono::milliseconds(2000));
    robot.jog_stop(-1);
    std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    robot.jog_ext(0,2,-1,0);
    std::this_thread::sleep_for(std::chrono::milliseconds(2000));
    robot.jog_stop(-1);

    robot.set_auto_manual_mode(true);
    std::this_thread::sleep_for(std::chrono::milliseconds(1000));
}
void move_ext()
{
    std::cout << "单独外部轴运动" << std::endl;
    MultiMovInfoList move_ext;
    move_ext.count = 1;
    move_ext.is_block = 1;
    move_ext.multi_mov_info[0].motion_unit_type = 1;
    move_ext.multi_mov_info[0].move_type = 0;
    move_ext.multi_mov_info[0].motion_unit_id = 0;
    move_ext.multi_mov_info[0].end_pos[0] = 100;
    move_ext.multi_mov_info[0].move_mode = 0;
    move_ext.multi_mov_info[0].j_vel = 80;
    move_ext.multi_mov_info[0].j_acc = 80;
    move_ext.multi_mov_info[0].radius = 0.0;
    ret = robot.multi_mov_with_ext(move_ext);
    std::cout << "单独外部轴运动结束1" << std::endl;
    move_ext.multi_mov_info[0].end_pos[0] = 200;
    ret = robot.multi_mov_with_ext(move_ext);
    std::cout << "单独外部轴运动结束2" << std::endl;
    move_ext.multi_mov_info[0].end_pos[0] = 0;
    ret = robot.multi_mov_with_ext(move_ext);
    std::cout << "单独外部轴运动结束3" << std::endl;

}

void movej_with_ext()
{ 
    std::cout << "关节运动+外部轴运动" << std::endl;
    MultiMovInfoList movej_info;
    movej_info.count = 2;
    movej_info.is_block = 1;
    movej_info.multi_mov_info[0].motion_unit_type = 1;
    movej_info.multi_mov_info[0].move_type = 0;
    movej_info.multi_mov_info[0].motion_unit_id = 0;
    movej_info.multi_mov_info[0].end_pos[0] = 100;
    movej_info.multi_mov_info[0].move_mode = 0;
    movej_info.multi_mov_info[0].j_vel = 80;
    movej_info.multi_mov_info[0].j_acc = 80;
    movej_info.multi_mov_info[0].radius = 0.0;

    movej_info.multi_mov_info[1].motion_unit_type = 0;
    movej_info.multi_mov_info[1].move_type = 0;
    movej_info.multi_mov_info[1].motion_unit_id = 0;
    movej_info.multi_mov_info[1].end_pos[0] = deg2rad(90);
    movej_info.multi_mov_info[1].end_pos[1] = deg2rad(90);
    movej_info.multi_mov_info[1].end_pos[2] = deg2rad(90);
    movej_info.multi_mov_info[1].end_pos[3] = deg2rad(90);
    movej_info.multi_mov_info[1].end_pos[4] = deg2rad(-90);
    movej_info.multi_mov_info[1].end_pos[5] = 0;
    movej_info.multi_mov_info[1].move_mode = 0;
    movej_info.multi_mov_info[1].j_vel = deg2rad(80);
    movej_info.multi_mov_info[1].j_acc = deg2rad(200);
    movej_info.multi_mov_info[1].radius = 0.0;
    ret = robot.multi_mov_with_ext(movej_info);
    std::cout << "关节运动+外部轴运动结束1" << std::endl;


    movej_info.multi_mov_info[0].end_pos[0] = 0;
    movej_info.multi_mov_info[1].end_pos[0] = deg2rad(0);
    ret = robot.multi_mov_with_ext(movej_info);
    std::cout << "关节运动+外部轴运动结束2" << std::endl;
}

void movel_with_ext()
{ 
    std::cout << "直线运动+外部轴运动" << std::endl;
    MultiMovInfoList multi_mov_info_list;
    multi_mov_info_list.count = 2;
    multi_mov_info_list.is_block = TRUE;

    multi_mov_info_list.multi_mov_info[0].motion_unit_type = 1;
    multi_mov_info_list.multi_mov_info[0].move_type = 0;
    multi_mov_info_list.multi_mov_info[0].motion_unit_id = 0;
    multi_mov_info_list.multi_mov_info[0].end_pos[0] = 100;
    multi_mov_info_list.multi_mov_info[0].vel = 80;
    multi_mov_info_list.multi_mov_info[0].acc = 80;

    multi_mov_info_list.multi_mov_info[1].motion_unit_type = 0;
    multi_mov_info_list.multi_mov_info[1].move_type = 1;
    multi_mov_info_list.multi_mov_info[1].motion_unit_id = 0;
    multi_mov_info_list.multi_mov_info[1].end_pos[0] = 0;
    multi_mov_info_list.multi_mov_info[1].end_pos[1] = -300;
    multi_mov_info_list.multi_mov_info[1].end_pos[2] = 300;
    multi_mov_info_list.multi_mov_info[1].end_pos[3] = deg2rad(-180);
    multi_mov_info_list.multi_mov_info[1].end_pos[4] = deg2rad(0);
    multi_mov_info_list.multi_mov_info[1].end_pos[5] = deg2rad(180);
    multi_mov_info_list.multi_mov_info[1].vel = 80;
    multi_mov_info_list.multi_mov_info[1].acc = 80;
    multi_mov_info_list.multi_mov_info[1].ori_vel = deg2rad(180);
    multi_mov_info_list.multi_mov_info[1].ori_acc = deg2rad(720);

    ret = robot.multi_mov_with_ext(multi_mov_info_list);
    std::cout << "直线运动+外部轴运动结束1" << std::endl;

    multi_mov_info_list.multi_mov_info[0].end_pos[0] = 0;
    multi_mov_info_list.multi_mov_info[1].end_pos[0] = -300;
    multi_mov_info_list.multi_mov_info[1].end_pos[1] = 100;
    multi_mov_info_list.multi_mov_info[1].end_pos[2] = 300; 
    ret = robot.multi_mov_with_ext(multi_mov_info_list);
    std::cout << "直线运动+外部轴运动结束2" << std::endl;

}

void movec_with_ext()
{ 
    std::cout << "圆弧运动+外部轴运动" << std::endl;
    MultiMovInfoList multi_mov_info_list;
    multi_mov_info_list.count = 2;
    multi_mov_info_list.is_block = TRUE;
    multi_mov_info_list.multi_mov_info[0].motion_unit_type = 1;
    multi_mov_info_list.multi_mov_info[0].move_type = 0;
    multi_mov_info_list.multi_mov_info[0].motion_unit_id = 0;
    multi_mov_info_list.multi_mov_info[0].end_pos[0] = 100;
    multi_mov_info_list.multi_mov_info[0].vel = 80;
    multi_mov_info_list.multi_mov_info[0].acc = 80;

    multi_mov_info_list.multi_mov_info[1].motion_unit_type = 0;
    multi_mov_info_list.multi_mov_info[1].move_type = 2;
    multi_mov_info_list.multi_mov_info[1].motion_unit_id = 0;
    multi_mov_info_list.multi_mov_info[1].mid_pos.tran.x = -145.178;
    multi_mov_info_list.multi_mov_info[1].mid_pos.tran.y = 15.344;
    multi_mov_info_list.multi_mov_info[1].mid_pos.tran.z = 291.208;
    multi_mov_info_list.multi_mov_info[1].mid_pos.rpy.rx = deg2rad(179.943);
    multi_mov_info_list.multi_mov_info[1].mid_pos.rpy.ry = deg2rad(0);
    multi_mov_info_list.multi_mov_info[1].mid_pos.rpy.rz = deg2rad(89.694);

    multi_mov_info_list.multi_mov_info[1].end_pos[0] = -345.178;
    multi_mov_info_list.multi_mov_info[1].end_pos[1] = 15.344;
    multi_mov_info_list.multi_mov_info[1].end_pos[2] = 291.208;
    multi_mov_info_list.multi_mov_info[1].end_pos[3] = deg2rad(179.943);
    multi_mov_info_list.multi_mov_info[1].end_pos[4] = deg2rad(0);
    multi_mov_info_list.multi_mov_info[1].end_pos[5] = deg2rad(89.694);
    multi_mov_info_list.multi_mov_info[1].vel = 80;
    multi_mov_info_list.multi_mov_info[1].acc = 200;
    multi_mov_info_list.multi_mov_info[1].ori_vel = deg2rad(180);
    multi_mov_info_list.multi_mov_info[1].ori_acc = deg2rad(720);
    ret = robot.multi_mov_with_ext(multi_mov_info_list);
    std::cout << "圆弧运动+外部轴运动结束1" << std::endl;

    multi_mov_info_list.multi_mov_info[0].end_pos[0] = 0;
    multi_mov_info_list.multi_mov_info[1].end_pos[0] = -300;
    multi_mov_info_list.multi_mov_info[1].end_pos[1] = 100;
    multi_mov_info_list.multi_mov_info[1].end_pos[2] = 300;
    multi_mov_info_list.multi_mov_info[1].end_pos[3] = deg2rad(-180);
    multi_mov_info_list.multi_mov_info[1].end_pos[4] = deg2rad(0);
    multi_mov_info_list.multi_mov_info[1].end_pos[5] = deg2rad(180);
    ret = robot.multi_mov_with_ext(multi_mov_info_list);
    std::cout << "圆弧运动+外部轴运动结束2" << std::endl;

}
```

