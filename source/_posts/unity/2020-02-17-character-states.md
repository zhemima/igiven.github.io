---
title : "角色状态设计"
---







https://www.cnblogs.com/Renyi-Fan/p/9569090.html



https://www.gameres.com/797311.html





![这里写图片描述](../../assets/images/2020-02-17-character-states/20161103213426625)

```
public abstract class Attribute
        {
            public int Hp { get; set; }
            public int Mp { get; set; }
            public int HpTotal { get; set; }
            public int MpTotal { get; set; }

            private Attribute m_Parent;
            private string m_Name;
            public Attribute(int hp, int mp)
            {
                this.Hp = hp;
                this.Mp = mp;

            }

            public Attribute(int hp, int mp, string name) : this(hp, mp)
            {
                this.m_Name = name;
            }

            public abstract void Calc();
            public abstract int ChildCount();
            protected void SetParent(Attribute child)
            {
                child.m_Parent = this;
            }

            public Attribute GetParent()
            {
                return this.m_Parent;
            }

            public void Show()
            {
                Calc();
                Console.WriteLine(string.Format("{0} 属性为： HP {1} MP {2}",this.m_Name, this.HpTotal, this.MpTotal));
            }
        }

```





```
public class AttributeComponent:Attribute
        {
            private List<Attribute> m_AttributeList = new List<Attribute>();

            public AttributeComponent(int hp, int mp) : base(hp, mp)
            {
            }

            public AttributeComponent(int hp, int mp, string name) : base(hp, mp, name) { }

            public override void Calc()
            {
                this.HpTotal = this.Hp;
                this.MpTotal = this.Mp;

                foreach (Attribute item in m_AttributeList)
                {
                    //递归计算属性和
                    if (item.ChildCount() > 0)
                        item.Calc();

                    this.HpTotal += item.HpTotal;
                    this.MpTotal += item.MpTotal;
                }
            }

            public override int ChildCount()
            {
                return m_AttributeList.Count;
            }

            public void AddAttribute(Attribute attribute)
            {
                SetParent(attribute);
                this.m_AttributeList.Add(attribute);
            }

            public void RemoveAttribute(Attribute attribute)
            {
                m_AttributeList.Remove(attribute);
            }
        }

```

```

public void TestMethod1()
        {
            AttributeComponent basic = new AttributeComponent(50, 100, "Ali");
            AttributeComponent lvUp = new AttributeComponent(50, 100, "升级增加");
            AttributeComponent weapon = new AttributeComponent(10, 20, "无级弓");
            AttributeComponent weaponEnchanting = new AttributeComponent(5, 5, "附魔增加");

            basic.Show();
            Console.WriteLine("升级啦");
            Console.WriteLine("---------------------------------------------------");
            lvUp.Show();
            basic.AddAttribute(lvUp);
            basic.Show();
            Console.WriteLine();
            Console.WriteLine("装备了武器");
            Console.WriteLine("---------------------------------------------------");
            weapon.Show();
            basic.AddAttribute(weapon);
            basic.Show();
            Console.WriteLine();
            Console.WriteLine("武器附魔");
            Console.WriteLine("---------------------------------------------------");
            weaponEnchanting.Show();
            weapon.AddAttribute(weaponEnchanting);
            weapon.Show();
            basic.Show();
            Console.WriteLine();
            Console.WriteLine("卸载装备");
            Console.WriteLine("---------------------------------------------------");
            basic.RemoveAttribute(weapon);
            basic.Show();
        }

```

https://blog.csdn.net/alistair_chow/article/details/53029909



https://blog.csdn.net/alistair_chow/article/details/53026375