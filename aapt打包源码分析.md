android源码tools/aapt目录
入口Main.cpp
else if (argv[1][0] == 'p')
  bundle.setCommand(kCommandPackage);

其中Bundle类在Bundle.h中
/*
 * Bundle of goodies, including everything specified on the command line.
 */


main方法最后调用result = handleCommand(&bundle);

case kCommandPackage:      return doPackage(bundle);
Main.h中声明extern int doPackage(Bundle* bundle);

Command.cpp
/*
 * Package up an asset directory and associated application files.
 */
int doPackage(Bundle* bundle)

// Load the assets.
assets = new AaptAssets();

err = assets->slurpFromArgs(bundle);
遍历所有的resources目录，比如drawable，values等
/*
* If a directory of resource-specific assets was supplied, slurp 'em up.
*/
for (size_t i=0; i<dirCount; i++) {
    const char *res = resDirs[i];
    count = current->slurpResourceTree(bundle, String8(res));

ssize_t AaptAssets::slurpResourceTree(Bundle* bundle, const String8& srcDir)

while (1) {
    struct dirent* entry = readdir(dir);

    String8 subdirName(srcDir);
    subdirName.appendPath(entry->d_name);

    AaptGroupEntry group;
    String8 resType;
    bool b = group.initFromDirName(entry->d_name, &resType);
}
这里注意
group.initFromDirName

bool
AaptGroupEntry::initFromDirName(const char* dir, String8* resType)
{
    const char* q = strchr(dir, '-');
    size_t typeLen;
    if (q != NULL) {
        typeLen = q - dir;
    } else {
        typeLen = strlen(dir);
    }

    String8 type(dir, typeLen);
    if (!isValidResourceType(type)) {
        return false;
    }

    if (q != NULL) {
        if (!AaptConfig::parse(String8(q + 1), &mParams)) {
            return false;
        }
    }

    *resType = type;
    return true;
}
可以看到，就是把dir中-以及后面的部分去掉，所以drawable,drawable-hdpi,drawable-mdpi等等就是属于一个group了。


call Resource::buildResources


Resource.cpp

buildResources
  -> status_t err = parsePackage(bundle, assets, androidManifestFile);
    parsePackage 获取manifest上的package属性，获取use-sdk指定的minSdk

  ResourceTable::PackageType packageType = ResourceTable::App;
  //构造resourceTable对象
  ResourceTable table(bundle, String16(assets->getPackage()), packageType);

ResourceTable.cpp

ResourceTable::ResourceTable(Bundle* bundle, const String16& assetsPackage, ResourceTable::PackageType type)

ssize_t packageId = -1;
    switch (mPackageType) {
        case App:
        case AppFeature:
            packageId = 0x7f;
            break;

        case System:
            packageId = 0x01;
            break;

        case SharedLibrary:
            packageId = 0x00;
            break;

        default:
            assert(0);
            break;
    }
如果你看过编译生成的R.java的代码，就会发现我们自己的app的资源的id都是以0x7f开头，系统的资源都是以0x01开头，其实就是这段代码在起作用。

sp<Package> package = new Package(mAssetsPackage, packageId);
    mPackages.add(assetsPackage, package);
    mOrderedPackages.add(package);
创建一个Package对象，并添加到集合mPackages中。
Package是在ResourceTable.h中定义的一个类
class Package : public RefBase {
    public:
        Package(const String16& name, size_t packageId);


回到buildResource方法，继续往下
err = table.addIncludedResources(bundle, assets);
用来添加引入的第三方的资源文件

  ResourceTable.cpp status_t ResourceTable::addIncludedResources(Bundle* bundle, const sp<AaptAssets>& assets)
  status_t err = assets->buildIncludedResources(bundle);

    AaptAssets.cpp
    status_t AaptAssets::buildIncludedResources(Bundle* bundle)
    通过调用mIncludedAssets.addAssetPath(includes[i], NULL)，来完成添加第三方的资源。
    在AaptAssets.h中定义了AssetManager mIncludedAssets;
    方法结束后，mHaveIncludedAssets = true;

  回到ResourceTable的addIncludedResources方法，继续往下
  mTypeIdOffset = findLargestTypeIdForPackage(assets->getIncludedResources(), mAssetsPackage);

  就是找到一个最大的type id作为基数，后面的id都会在这个基础上累加1

回到Resource.cpp的buildResources方法，继续往下，
collect_files(assets, resources);该方法主要就是收集资源文件的类型，跟新传入的resources

//将收集到的resources赋值给assets对象
assets->setResources(resources);

sp<ResourceTypeSet> drawables;
ASSIGN_IT(drawable);
找到drawable类型
遍历xml，indent是name属性
err = parseAndAddEntry(bundle, in, &block, curParams, myPackage, curType, ident,
                        *curTag, curIsStyled, curFormat, curIsFormatted,
                        product, NO_PSEUDOLOCALIZATION, overwrite, &skippedResourceNames, outTable);
