# what is　considered a clean iOS architecture? 

In a nutshell 

an architecture or a design 

**allows the team to develop maintain test and extend the features of the app doing all this easily in the short and long term**

it's not just about the present it's about the future can the architecture sustain future requirements come in welcome future requirements can it sustain deletions if you want to remove parts of the system 

**allows extensibility of the system as adding new features or even removing features or changing features**

so design is only considered good or clean when you look at it in a timeline of changes if it is easy to change the system if it is easy to maintain a constant pace of changes throughout the development cycle you have a good design

**makes change easy**
it's not only about the current design the current features but will be sustained the design we will sustain the product in the development of the app throughout its lifecycle 

**allows the team to use the resources smartly**

for example it should be easy to add more people in the team and increase the productivity of the team but if every time you add more people you make things harder you don't have a good design as well because it cannot allocate resources as needed


# What are the desired traits of a clean iOS architecture? 

you should **welcome requirement changes which means it should make it easy to maintain a constant and fast pace** 

when you start a project six months after the project started one year later two years later you should maintain a fast pace so if a feature took a week to develop in the beginning of the project a year later a similar feature should take a week or less but if a feature with a similar complexity takes three weeks now after a year of development it means that your architecture is getting in the way

exactly so a clean architecture a good architecture will make it easier to maintain a fast in constant pace 

another very important trait is **testing** and **making the testing of the app easier** right so what good architecture will

make testing easier that's exactly right because if you can't test your components this signifies that there is tight coupling there are implicit dependencies and overall rigidity in your system so as a result testing would be a nightmare

**allows independent development it allows developers to work in separate features without getting in each other's way**

which means it can bring more people in the team and increase productivity of the team this is the reason why the team a year from now can yield the same results productivity wise it's because it's gonna enable the team to develop independently this means to develop in parallel this means two three four X the results that you could have instead of working as a unison so this is quite remarkable to achieve and as you said in the beginning of a project it's easy to see fast results but as we add more code in the codebase things get harder so a

clean architecture promotes this independent development this parallel development and this is what you get at the end you get features delivered continuously and developed very very fast so things will get harder as time passes exactly it should be as easy to add a feature in the beginning of a project and a year later yes or two years later in three years later I can argue that it might be even easier afterwards a year later or two years later because you already have the whole structure you know how the system is being built in the beginning


perhaps you are a bit more cautious you're not sure how this project will plan a half so yeah it might be that it can be even easier down the road exactly
so if you can maintain a constant and fast pace it means **you can estimate more accurately when features will get done** because you don't need to account for things that will get in your way of delivering a future the design is clean enough that you can think of the feature as an isolate change so it's easier to estimate that change but if every time you need to add a new feature or change a feature you'll need to affect multiple modules of your system it means they don't have a good design a good architecture so this change can cascade in a way that it's gonna take double triple the time that's exactly right and predictions and time and estimates affect the business so we talked a bit

about how a clean architecture affects developers the development team but what about the business and these are some of the very very positive side effects of a clean iOS architecture that the **business will receive more accurate estimates** they're gonna be able to plan better their vision and if the business can plan better division they can achieve better results and everyone wins exactly because if the business grows the team grows there's more investment yes and everyone wins yes so 


# how can I design a clean iOS architecture for my iOS app?

well it depends on your app but mainly you need to **decouple high level policies or business rules from infrastructure details like networking UI databases and the composition of the application** as well


so need to decouple those things into separate layers so separating high-level policies from low-level details at a feature level and the application level

as well as needed you need to balance

the trade-offs of the decoupling does you want low coupling and high cohesion right so 

for example organizing your project into layers business logic layer networking layer UI layer composition layer and etc breaking down the problem into small problems so can solve one challenge at a time you can test things in isolation you can change things in


**isolation can extend things in isolation** 

so you can add new behavior without affecting existing behavior which makes your system or open closed open for extension and closed for modification which means the UI will not hold business logic and a business project will not hold UI logic right just like

the UI will not depend on API or networking specific details and the networking layer will not depend on UI specific code or details so you can change things in isolation you can develop them in isolation can develop them in parallel we can test them in

isolation can extend them in isolation without affecting other layers under modules and then you can choose how to slice your features in your layers maybe horizontally or vertically depends on the problem at hand but a clean architecture will allow you to modify the slices of your application as needed that's exactly correct it's not how you're going to separate your application and what kind of modules you're gonna have is more the plumbing between the modules how they're going to communicate between them and how they will remain very consistent in their goal 


for example as you mentioned you have a UI module and you don't want Beast rules inside the UI module you don't want any services or service related

work like networking persistence perhaps　it's exactly the same for the other way　around and why don't you want this is　logic in the UI layer because you don't　want to risk changing a UI detail in　breaking some business logic some　business rules you also want to decouple　it so you can change your user interface　without having to rewrite the business　rules　it can even compose different user interfaces with the same business logic　so can we use the business logic with　different UIs we can have a UI kiti UI　for iOS nap kit UI for Mac OS a watchkit　UI for the watch and you are reusing the　business logic so you know we have a　concise place where the business object lives so every application will behave accordingly 

imagine if the business

comes in and tells you that they're

planning to do a reskin of the

application say we're not gonna change

anything in the backend any business

rules we just want to have this new

layout because we believe it's gonna

convert better yes and the iOS team says

well we're gonna need six months to do

so yes that's problematic right there or

a year or two years to do so or maybe

they're going to say we need to rewrite

the entire application so if this

happens they'll have a conflict of

interest with the business because on

the business side they have a reasonable

expectation the change in the user

interface should be easy right should be

just a matter of plugging in a new user

interface with the same logic so that's

the expectation the businesses and

customers have towards the application

and we as developers need to fulfill

that expectation because it is possible

if you decouple your modules you can

plug in any user interface easily

exactly so it changed such as changing

the user interface or changing the API

layer should not demand a rewrite if it

does you don't have a good and clean

architecture and it's gonna cost

everyone a lot of time

money 

# what practices and patterns support a clean iOS architecture?

well the application of the solid principles even have a podcast about that podcast number five good application of dependency injection they're interested watch or listen to the podcast three and number fifteen also well stablished design patents we

talked about it in podcast fourteen high

test coverage with fast and reliable

tests because writing tests following

TDD will exert incentives for the team

to think about design more often you

write the test first you need to think

about the design so the test will exert

some pressure on design thinking which

is super important very interested gonna

learn more podcast number one and number

13 we will talk about testing strategies

and test-driven development so combining

that's driven development solid

principles good design principles domain

driven design dependency injection it

will yield good results in the short and

long term 



but it requires discipline it's a constant practice in to apply it over and over there is no way to create a clean architecture and think you are done the architecture is changing constantly it needs to evolve it's a constant process need to be improving it

all the time as there are new

requirements you need to reevaluate the

decisions so far and see how those on

your requirements fit and how you can

facilitate the implementation of those

new requirements so you might have to

adapt the architecture a little bit and

that should be easy to do so if you

create tiny composable and decoupled

components you are going to facilitate

those changes that's why you follow

principles that's why you follow

practices like TDD as well

exactly and the point is long-term if

you want continuously to see good

results you need all these principles

and all these practices otherwise for

the short term you don't need any

listings if you show something on the

screen and the business is happy with it

that's it you have it but what happens a

month from now two months from now well

that's the problem there if you don't do

these practices chances are things will

get harder and harder and harder and at

some point you're gonna have to stop

development to fix previous bugs and

defects and at some point perhaps you're

gonna stop development altogether for

the current version or for the current

application you're gonna have to rewrite

the whole application so if you have a

simple application they're gonna develop

it and discard in two months anything

you do is fine it probably don't need a

clean architecture yes

but hopefully you are working on bigger

challenges developing applications for

the long term for profitability for

growth so the creating software for the

long term you need clean architecture

otherwise we're gonna have to rewrite

the app at some point 


# what is the goal of cleaning iOS x texture

why should I bother okay well 

the goal is to reduce cost of change make change easy and increase the rewards in the

short and long term that's it so if you

are in the game for the long term for

developing software that you want it to

succeed now and in the future and as you

add more features to it you need to

bother about cleaning iOS architecture

create good designs good solutions

otherwise if you plan to develop an app

and throw it away in two months that's

fine whatever you do is fine that's

short-term thinking the high-paying

challenges are the ones for the long

term

so we recommend you to bother about

canary architecture if you want a

remarkable career if you want to work in

interesting and exciting projects that's

exactly right

and we have a quote here by Robert C

Martin in his book clean architecture

and Herbert C Martin says the measure of

design quality is simply the measure of

the effort required to meet the needs of

the customer if that effort is low and

stays low throughout the lifetime of the

system the design is good if that effort

grows with each new release the design

is bad it's simple as that

simple as that yeah that's it these are

the dynamics we've been talking about in

the beginning the development is fast as

you progress if you don't use good

principles quit practices it's gonna

start slowing down now what does it mean

to slow down because maybe for

developers it's not clear or they don't

care but it means the cost increases

every day every minute that the business

can't materialize the vision then their

cost of operation increases so the goal

is to work with the business to achieve

its goals the business vision yes the

business achieved their goals they can

grow and everyone in that business

benefits and the customers benefits and

the developers will also benefit of that

growth so think of the actors in this

relationship in this process the

customers want a reliable and delightful

experience with the applications they

interact with or the services they are

even paying for and the business wants

to keep materializing their vision

delivering those features to their

customers continuously they want to

deliver good and profitable products and

the iOS team wants to collaborate

effectively to fulfill the business

vision and deliver great products to the

customers and of course keep improving

as professionals and getting well

compensated for their hard work everyone

can benefit if they align their

incentives so we as developers it's our

responsibility to deliver good solutions

for the short and long term and that's

where clean architecture fits is going

to help us keep materializing the

business vision and deliver

great products to the customers so we

can keep progressing as professionals in

getting high rewards for our hard work

simple as that



next question how can I evaluate my iOS

app architecture well ideally you need

some metrics for example productivity is

the team as productive now as they were

a year ago or a month ago or a week ago

if things takes longer now than a month

ago something is getting in the way

maybe it's the architecture maybe it's

the system design that's getting in the

way

this is an indicator that maybe the

design is getting the way of the

developers as the system is getting

harder and harder to change the team

gets his lower and slower you can also

ask your peers are you happy with the

current design do you find it easy to

deal with this codebase then you need to

evaluate the responses maybe they're not

happy not because the design is not good

but because they don't understand the

design maybe they need some training

they need some support

maybe pairing more with the developers

will help them understand the design if

they have experienced they understand

the design and they just hate it they

are not happy with the design they know

it's getting their way well that's what

my trackers should assess and find a

solution to improve the design to make

things easier for the team thus

increasing productivity in the whole

business under metrics modularity in the

codebase is it flexible is it easy to

add a new feature does it affect

multiple modules is an easy change easy

and is a hard change possible can we add

features easily can we extend features

easily can we remove a feature easily

can we replace a framework easily can we

replace like a vendor for using realm as

a persistence framework but now the

business decided to use another one or

there's a new version that is better but

they have different API see how easy it

is to replace the infrastructure details

it should be easy if it's not there's

some coupling crossing modules another

metric is testability and regressions

how many regressions you had in the past

year as

mom this week is it increasing why quite

a question why just maybe the team is

not testing the code then it's are

having more regressions but not because

they don't want a test it's just because

it's too hard to test with the current

design that's why you need to assess how

easy it is to test the codebase because

if it's hard the developers will not

test the code regressions will go up so

it should be easy to test the code and

maintainability do you need big

refactorings often you need to stop

development to perform refactorings

because ideally with factoring should be

part of the process every time you are

adding a new feature or changing some

features it should be part of the

process to refactor it you shouldn't

have to schedule refactoring

it should be part of the process but if

you need bigger effect earnings often

you don't have a good design right all

the team doesn't have this skills to

maintain the codebase maybe they need

training so you need to assess all those

metrics together because the codebase is

evolving constantly every time you add a

new feature every new commit can change

the code base you can make things more

coupled or you can introduce a

regression you can make testing harder

in the future so every tiny change can

affect the system architecture so it's a

constant evaluation process everyone in

the team should be conscious about it

and if they are not they need training

so to evaluate the app architecture need

to check the metrics there is impacting

the team productivity and all these

metrics are just connected productivity

is a computed metric if you have for

example good results in your technical

metrics then most probably productivity

is gonna be good but if you have

terrible results in your technical

metrics then in productivity will suffer

guaranteed there is absolutely no way as

time progresses the productivity is

gonna remain high or even the same not

even high yes

it's a constant effort it is and these

metrics need

monitoring and not just these metrics

because these are high level metrics but

other metrics in the code base as well

we mentioned in the past about build

times or test times these are good

indicators they're not like the most

fancy ones but they are good indicators

because as more code is being poured in

your codebase then these indicators with

either suffer or the team will find good

ways to support the new editions

thus the operation will remain fast will

remain smooth yeah how often do you

deliver them to deliver every six months

can you cut this by half can you deliver

every three months and then can you cut

it by half again until you are releasing

frequently maybe weekly that's the level

productivity you can achieve with a

clean architecture in a productive team

so you need both the technical skills

and the constant effort of applying them

with discipline yes next question how

does MVC mvvm or MVP feat in a clean iOS

architecture and which one should I use

right well they all fit in the user

interface layer right if you're breaking

down your application in modules and

layers MVC mvvm and MVP leaves in the UI

layer and that's it the application have

many other layers so you cannot say that

your architecture is MVC mvvm or NDP no

your user interface is using MVC MDM or

MVP to organize a project yes they are

just patterns and guidelines to organize

the user interface part of your

application right and that's it so you

can use any of those patterns all of

them support a clean architecture when

used in the user interface which is what

they were meant to be doing exactly

because they are isolated or other they

should be isolated that's it so you can

use MVC mvvm MVP and achieve a clean

Airways architecture you should check

out episode 12 for more on these design

patterns that's it

next question what is an iOS architect

and how can I become one well an iOS

architect is a developer responsible for

analyzing and refining the code base

design repeatedly continuously so an iOS

architect should be making decisions to

facilitate or accommodate change in the

codebase making design decisions to make

things easier to change to manage to

test but most teams don't have someone

with the title of iOS architect and

every tiny change in the codebase

affects the architecture so everyone

that contributes code to the codebase

needs to be conscious about design about

how their changes affect the overall

architecture if you have an iOS

architect in your team or not

right if you have an iOS architect you

will have someone they will be paying

more attention to this and helping the

team deliver better results but

regardless everyone in the team is

responsible for the system architecture

everyone the contributes code is

affecting the architecture so somehow we

are all architects that's it do you come

in code you're responsible for the

architecture of the system are you the

architect no there is no the architects

even if there is our specialized role

for checking and setting the

architecture of the system everyone is

responsible for their contributions

because any commit can influence the

whole up the apps architecture so

ideally an architect should also be

contributing to the codebase because if

the architect just stays at the

whiteboard designing diagrams and

high-level guidelines for the codebase

then how can they guarantee that the

team is actually implementing that

design right so ideally there is an

architect guiding the team they should

be pairing with the team it should be

teaching the team how to create those

flexible solutions those clean solutions

so they don't become a bottleneck as

well there shouldn't be one person only

responsible for the design everyone in

the team is responsible for the design

in the architecture so maybe a better

question is not how can I become an iOS

architect because this

is rare a better question should be how

do I become good at creating clean

solutions for my iOS code bases yes how

can I create solutions to help the team

be more productive that can help the

business deliver their vision

perpetually and how can we deliver

better solutions faster to our customers

no regressions no bugs how can i

facilitate testing to prevent

regressions and bugs and so on right and

for that to happen it's vital for the

team to understand the product roadmap

yes what does the backlog look like yes

if you want a sustainable pace in the

short and long term you need to

understand where this product is going

so need to understand the roadmap that's

how you can find solutions that can

solve the problem now and also make the

extensibility of the system easy or

easier so yeah you need to know and

understand the product roadmap you also

need to understand this skills in the

team right do they need more training

do you need more training maybe you need

to go to your boss and ask hey if you

want to fulfill this product roadmap we

need better skills in the team we need

some training and every business should

train their staff should train their

developers to train everyone that's the

responsibility of the business they need

to provide the training for you to

outperform in your role if not maybe

should find a business that cares about

the long term because you want to be

surrounded by people that want growth so

everyone keeps growing together yes

because what's good about trying to set

good standards for a clean iris

architecture but the team doesn't have

the skills to perform

dependency injection for example or

write tests first and test components in

isolation yes if the team don't have the

skills and the business is not investing

in them training them how can they

deliver sustainable solutions that's not

possible

yes so the team lead should assess that

and find the resources to train the team

that meaning time money and space for

them to

try what they learn to implement it so

if you want to become a good iOS

architect you need to understand the

product roadmap need to understand the

team skills and help them improve if

necessary and finally you need to

understand the current state of the

codebase where do you stand

are you in good shape bad shape do you

need to improve it you need to keep

asking those questions all the time

because the product roadmap will change

you need to understand those changes the

team skills will change people who join

people will leave and you also need to

understand the current state of the

codebase and where it's heading if you

want to out perform out deliver and

achieve a remarkable career you should

be concerned about those challenges

that's how you deliver value to the

business to the customers and your help

the business grow and the team grow as

well next question how can I learn how

to design a clean architecture for my

iOS apps

first of all there is no one market

texture that solves all the problems

this is a continuous process and it

should be refining it as you go so how

do you learn how to design clean

architectures how do you learn how to

refine the design over and over well you

need to learn practice and execute

continuously so need to learn from books

courses mentors need to practice

practice and practice practice with some

personal projects practice at work and

execute keep executing with our own

projects or at work and keep delivering

apps that's how you gonna learn I wanna

learn fast you need to learn from others

yes if you are in a team with good

senior developers lead developers they

can teach you

architects they can teach you fantastic

just learn the most you can from them

pair with them go for a coffee and ask

bazillion questions gonna learn fast

we're gonna learn from others but

ultimately you need to practice you

don't practice by listening to a podcast

they're not practicing by watching a

video you need to pair

exactly and or we're teaching the Irish

laid essentials course is that creating

a clean iOS architecture is a computed

process he depends on practices and

principles that you will continuously

relentlessly apply to your project we're

talking about modular design we're

talking about testing first applying the

solid principles having good dependency

inversion with third-party libraries so

separating high-level policies from

low-level details or infrastructure

details that's it it's almost inevitable

if you apply these principles and practices for not to end with something

it's gonna be considered a clean iOS architecture and you don't need to get it right at first yes exactly

if you develop a clean flexible design

even if you get it wrong it's easy to

change it to improve it it's not rigid

yes so creating the perfect architecture

out of the blue is an unrealistic

expectation because the architecture

will keep evolving you will never be

perfect but it needs to be clean needs

to be flexible so you can keep evolving

it that's the thing there is no single

blog post or video or podcast or course

or book or Orofino that's gonna say okay

this is it even if they do say that it's

not sustainable because things will

change in the future and then you don't

have written down the solution for the

problem that you're gonna have in the

future so what do you do well that's why

we're saying all the time a little

practice executes continuously because

you need to gain the experience and the

knowledge to navigate in the darkness

basically when you don't know what the

problem is and what the solution you

need to have a process to find that

guidelines processes and practices yes

yes they've been using good results over

and over

it's been documented in books courses

from more experienced people that tried

out many different solutions so stand on

the shoulder of giants learned from

others that's it

so don't need to reinvent the wheel but

everyone will be dealing with different

challenges you need a solid foundation

of the design principles so you can

solve your own problems very interested

we have a free series on YouTube with

over 30 coding sessions and you can find

a link in the show notes 


next question

the app I'm working on is a mess 

# how can I be factor a legacy iOS project into a clean architecture 

well that's not gonna

be easy a messy legacy project won't be

cleaned in a day in a week if you're

working in a team this is a team effort

it's not the responsibility of one

developer even if you have an iOS

architect that's not their

responsibility everyone's responsibility

to improve the design and make things

better there is no way a project like

this can be cleaned from one person it

must be a team effort otherwise there's

gonna be conflict all day long yes

conflict low morale one person fixes a

part of the codebase the other one

breaks it yes you know there will not be

very nice to be part of yes so needs to

be a team effort now if you have this

massive messy legacy code base you also

cannot stop development and put this

refactoring on a schedule and expect

your boss to accept it right because

they don't want to stop the development

yes they expected this codebase to

deliver their vision perpetually

stopping the development to refactor

everything is not a good solution

because it's going to be very hard to

get buy-in from your managers and bosses

what do you do instead you fix things

you improve things as you go yes you get

everyone in the team to improve things

as you go so there is no conflict of

interest and you don't need to stop

developing new features if something is

working leave it there it's working we

need to change a part of the system or

add a new feature new code should be

clean should be tested and follow good

design principles and if you're changing

code they already exists you clean up

just that part

you're changing so as you keep changing

and improving things at some point you

will clean up the whole codebase and

this may take a year don't expect it to

happen in a couple of weeks no you're

gonna be improving things as you go as a

team effort so you need to find that

strategy with your team to do things as

you go to improve the code base as you

go and it is possible you can start by

reading they're working effectively with

legacy code by micro feathers a classic

or or even further and get some training

every business that cares about

realizing their vision perpetually they

know they need skills in the team they

will be happy to invest in skills if

they care about growth that's it the key

word is invest these are not costs this

is an investment in the business we're

writing the code base is a cost

investing in skills is an investment yes

that's it so if you want to improve the

design of a legacy code base you don't

need to rewrite it from scratch improve

things as you go as a team effort and

that's it for today don't forget to

check the show notes where you can find

a link to our free coding sessions where

you can learn TDD modular design in

clean architecture and if you want to go

one step further

join us at academy dot essential

developer.com let us know your thoughts

your comments your feedback and we'll

see you again next time bye your Thea

[Music]

